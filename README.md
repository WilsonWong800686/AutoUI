#!/usr/bin/env python
# -*- coding: utf-8 -*-

"""
阴阳师自动化脚本 - 增强版 (简化UI版本)
作者: AutoClick团队
版本: 1.0.0 (稳定版)

功能特点:
1. 图形用户界面 - Windows风格设计，美观易用
2. 多设备管理 - 同时控制多个模拟器实例
3. 增强的设备搜索功能 - 自动搜索并连接MuMu模拟器
4. 自动化按钮点击 - 基于OpenCV模板匹配实现按钮识别
5. 实时状态监控 - 显示每个设备的运行状态和进度
6. 配置持久化 - 记住窗口位置和应用设置

使用方法:
1. 点击"刷新设备"以搜索可用的模拟器实例
2. 点击"开始任务"配置运行时间和设备
3. 系统将自动在游戏中寻找并点击相应按钮
"""

import os
import sys
import time
import threading
import subprocess
import logging
import tkinter as tk
from tkinter import ttk, messagebox, scrolledtext, font
from typing import List, Tuple, Dict
import re
import traceback
import queue
import random
import json

# 设置日志记录
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - [%(threadName)s] - %(message)s'
)
logger = logging.getLogger(__name__)

# 设置消息队列，用于线程间通信 - 避免直接从后台线程修改UI
message_queue = queue.Queue()

# 配置文件路径 - 用于存储用户设置和窗口位置
CONFIG_FILE = "app_settings.json"

# 状态日志记录器 - 提供线程安全的日志管理
class StatusLogger:
    def __init__(self):
        self.callbacks = []       # 回调函数列表，用于通知UI更新
        self.history = []         # 日志历史记录
        self.log_lock = threading.Lock()  # 线程锁确保日志操作的原子性
        
    def log(self, device, message, level="info"):
        """
        记录日志并通知所有注册的回调函数
        
        参数:
            device: 设备标识符
            message: 日志消息内容
            level: 日志级别 (info, warning, error)
        """
        with self.log_lock:  # 使用锁确保日志操作的原子性
            log_entry = {
                "time": time.strftime("%H:%M:%S"),
                "device": device,
                "message": message,
                "level": level
            }
            self.history.append(log_entry)
            
            # 保持历史记录不超过1000条，避免内存泄漏
            if len(self.history) > 1000:
                self.history = self.history[-1000:]
            
            # 将日志事件放入队列，由主线程处理，确保线程安全
            message_queue.put(("log_event", log_entry))
                
            # 同时输出到标准日志系统，方便调试
            if level == "info":
                logger.info(f"[{device}] {message}")
            elif level == "warning":
                logger.warning(f"[{device}] {message}")
            elif level == "error":
                logger.error(f"[{device}] {message}")
    
    def register_callback(self, callback):
        """注册回调函数，当有新日志时调用"""
        with self.log_lock:
            self.callbacks.append(callback)
    
    def get_history(self, count=50):
        """获取最近的日志历史记录，用于显示在UI上"""
        with self.log_lock:
            return self.history[-count:]

# 全局状态日志记录器实例
status_logger = StatusLogger()

class AutoClickUI:
    """
    图形用户界面主类 - 管理整个应用程序的UI和交互逻辑
    
    负责:
    1. 创建并管理窗口、控件和样式
    2. 处理用户输入和界面交互
    3. 管理设备线程和状态更新
    """
    def __init__(self, root):
        self.root = root
        self.root.title("阴阳师自动化工具 - 增强版 (稳定版)")
        
        # 加载配置 - 恢复先前的设置
        self.load_config()
        
        # 如果有保存的窗口位置和大小，则设置
        if self.config.get("window_geometry"):
            self.root.geometry(self.config["window_geometry"])
        else:
            self.root.geometry("800x600")  # 默认窗口大小
            
        # 防止窗口刷新时重复刷新设备
        self.is_refreshing = False
        
        # 监听窗口大小和位置变化，以便保存配置
        self.root.bind("<Configure>", self.save_window_geometry)
        
        # 捕获窗口关闭事件，确保正常关闭所有线程
        self.root.protocol("WM_DELETE_WINDOW", self.on_closing)
        
        # 设置颜色方案 - 定义UI风格
        self.setup_colors()
        
        # 设置全局字体 - 确保界面一致性
        self.setup_fonts()
        
        # 创建自定义样式 - 应用到各个控件
        self.setup_styles()
        
        # 注册状态回调 - 接收日志更新
        status_logger.register_callback(self.status_callback)
        
        # 设备管理 - 存储设备相关数据和线程
        self.auto_clicker = AutoClicker()  # 自动点击功能封装
        self.device_threads = {}   # 设备工作线程
        self.stop_events = {}      # 线程停止标志
        self.pause_events = {}     # 线程暂停标志
        self.device_frames = {}    # 设备UI卡片
        
        # 设置主框架 - 构建UI结构
        self.setup_ui()
        
        # 刷新设备列表 - 搜索并连接设备
        self.refresh_devices()
        
        # 定时器，每秒更新一次状态 - 更新设备信息
        self.update_timer()
        
        # 开始处理消息队列 - 处理后台线程发送的通知
        self.process_message_queue()
    
    def load_config(self):
        """
        加载应用程序配置
        
        从CONFIG_FILE指定的JSON文件中读取配置，如果文件不存在或读取失败，
        则使用空字典作为默认配置。配置包括窗口位置、大小和其他用户设置。
        """
        try:
            if os.path.exists(CONFIG_FILE):
                with open(CONFIG_FILE, 'r', encoding='utf-8') as f:
                    self.config = json.load(f)
            else:
                self.config = {}
        except Exception as e:
            logger.error(f"加载配置出错: {str(e)}")
            self.config = {}
    
    def save_config(self):
        """
        保存应用程序配置
        
        将当前配置保存到CONFIG_FILE指定的JSON文件中，包括窗口位置、
        大小和其他用户设置，确保在程序下次启动时能够恢复状态。
        """
        try:
            with open(CONFIG_FILE, 'w', encoding='utf-8') as f:
                json.dump(self.config, f, ensure_ascii=False, indent=2)
        except Exception as e:
            logger.error(f"保存配置出错: {str(e)}")
    
    def save_window_geometry(self, event=None):
        """
        保存窗口位置和大小
        
        当窗口大小或位置发生变化时调用，记录当前几何信息到配置中，
        但不立即保存到磁盘，而是在程序关闭时统一保存，以减少I/O操作。
        
        参数:
            event: Tkinter事件对象
        """
        if event and event.widget == self.root:
            try:
                geometry = self.root.geometry()
                self.config["window_geometry"] = geometry
                # 不需要每次都保存到磁盘，只在程序关闭时保存
            except Exception as e:
                logger.error(f"保存窗口位置出错: {str(e)}")
    
    def on_window_resize(self, event=None):
        """
        处理窗口大小变化事件
        
        当窗口大小改变时，使用防抖动技术确保不会频繁触发界面刷新，
        等待调整完成后再重新布局设备列表。
        
        参数:
            event: Tkinter事件对象
        """
        # 如果正在刷新设备，则跳过
        if self.is_refreshing:
            return
            
        # 仅处理来自根窗口的调整大小事件
        if event and event.widget == self.root and hasattr(self, 'auto_clicker') and self.auto_clicker.devices:
            # 使用防抖动技术，避免频繁触发刷新
            if hasattr(self, "_resize_timer_id"):
                self.root.after_cancel(self._resize_timer_id)
                
            # 设置定时器，在调整窗口大小停止500毫秒后再重新布局
            self._resize_timer_id = self.root.after(500, self._delayed_resize)
    
    def _delayed_resize(self):
        """
        延迟执行的调整大小处理函数
        
        当窗口大小调整停止一段时间后，检查窗口宽度是否发生显著变化，
        如果是，则重新刷新设备列表以适应新的窗口大小。
        """
        # 如果正在刷新设备，则跳过
        if self.is_refreshing:
            return
            
        # 检查窗口宽度是否显著变化
        current_width = self.root.winfo_width()
        if not hasattr(self, 'last_width') or abs(current_width - self.last_width) > 100:
            self.last_width = current_width
            # 更新设备列表布局
            self.refresh_devices()
    
    def setup_colors(self):
        """
        设置全局颜色方案 - Windows游戏风格
        
        定义应用程序使用的所有颜色常量，确保整个界面颜色一致，
        风格统一，提供良好的视觉体验。
        """
        # 游戏风格配色
        self.bg_color = "#EFEFEF"        # 背景色 (浅灰)
        self.frame_bg = "#FFFFFF"        # 框架背景 (白色)
        self.accent_color = "#007ACC"    # 强调色 (蓝色)
        self.button_bg = "#E1E1E1"       # 按钮背景色
        self.hover_color = "#D1D1D1"     # 悬停颜色
        self.header_bg = "#F0F0F0"       # 标题栏背景
        self.text_color = "#333333"      # 文字颜色
        self.border_color = "#CCCCCC"    # 边框颜色
        
    def setup_fonts(self):
        """
        设置全局字体 - Windows游戏风格
        
        创建应用程序使用的所有字体，并设置为默认选项，
        确保整个界面字体统一，提升用户体验。
        """
        # 创建自定义字体
        self.title_font = font.Font(family="微软雅黑", size=16, weight="bold")
        self.normal_font = font.Font(family="微软雅黑", size=10)
        self.button_font = font.Font(family="微软雅黑", size=10, weight="bold")
        self.log_font = font.Font(family="等线", size=9)
        
        # 设置默认字体
        self.root.option_add("*Font", self.normal_font)
        
    def setup_styles(self):
        """
        设置自定义样式 - Windows游戏风格
        
        使用ttk.Style配置各种控件的外观，包括背景色、前景色、字体等，
        使整个应用具有一致的Windows游戏风格。
        """
        self.style = ttk.Style()
        
        # 尝试使用Windows主题，提供更好的原生体验
        try:
            self.style.theme_use("vista")
        except:
            pass  # 如果不支持，继续使用默认主题
        
        # 配置全局样式
        self.style.configure(".", background=self.bg_color, foreground=self.text_color)
        
        # 配置标题标签样式
        self.style.configure("Title.TLabel", 
                             font=self.title_font, 
                             background=self.bg_color, 
                             foreground=self.text_color)
        
        # 主要按钮样式 - 所有按钮使用统一样式
        self.style.configure("TButton", 
                             font=self.button_font,
                             background=self.button_bg,
                             relief="raised",
                             borderwidth=1)
        
        # 为按钮添加悬停效果，增强交互感
        self.style.map("TButton",
                       background=[('active', self.hover_color)])
        
        # 框架样式
        self.style.configure("TFrame", background=self.bg_color)
        self.style.configure("Card.TFrame", 
                             background=self.frame_bg,
                             relief="solid", 
                             borderwidth=1)
        
        # 设备卡片标题栏
        self.style.configure("CardHeader.TFrame", 
                             background=self.header_bg,
                             relief="solid",
                             borderwidth=1)
        
        # 标签样式
        self.style.configure("TLabel", background=self.bg_color)
        self.style.configure("Card.TLabel", background=self.frame_bg)
        self.style.configure("Header.TLabel", background=self.header_bg, font=self.normal_font)
        
        # 进度条样式
        self.style.configure("TProgressbar", 
                             thickness=15,
                             background=self.accent_color,
                             troughcolor="#F5F5F5",
                             borderwidth=0)
        
        # 分隔线样式
        self.style.configure("TSeparator", background=self.border_color)
        
        # 设置窗口背景色
        self.root.configure(background=self.bg_color)
        
    def process_message_queue(self):
        """
        处理消息队列
        
        从队列中获取消息并处理，用于在主线程中安全地更新UI，
        避免多线程导致的UI更新问题。每100毫秒检查一次队列。
        """
        try:
            # 处理所有当前队列中的消息
            for _ in range(100):  # 限制每次处理的消息数量，避免阻塞主线程
                try:
                    message_type, data = message_queue.get_nowait()
                    
                    if message_type == "log_event":
                        self.update_status()
                    # 可以处理其他类型的消息...
                    
                    message_queue.task_done()
                except queue.Empty:
                    break
        except Exception as e:
            logger.error(f"处理消息队列出错: {str(e)}")
        
        # 安排下一次处理，确保持续监听
        self.root.after(100, self.process_message_queue)
        
    def setup_ui(self):
        """
        设置用户界面 - Windows游戏风格
        
        创建并布局应用程序的所有UI组件，包括标题栏、控制按钮、设备列表和日志区域。
        整体布局采用垂直方向的盒式布局，各组件从上到下依次排列。
        
        布局结构:
        1. 标题区域 - 显示应用名称
        2. 控制按钮区 - 包含主要操作按钮
        3. 设备列表区 - 显示已连接设备的卡片式界面
        4. 日志区域 - 显示操作日志
        5. 状态栏 - 显示当前状态信息
        """
        # 主框架 - 包含所有组件
        main_frame = ttk.Frame(self.root)
        main_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)
        
        # 标题和按钮区域 - 顶部控制栏
        header_frame = ttk.Frame(main_frame)
        header_frame.pack(fill=tk.X, pady=(0, 10))
        
        # 标题 - 应用名称
        ttk.Label(header_frame, text="阴阳师自动化工具 - 增强版", style="Title.TLabel").pack(side=tk.LEFT, pady=5)
        
        # 控制按钮区 - 主要功能按钮组
        control_frame = ttk.Frame(main_frame)
        control_frame.pack(fill=tk.X, pady=5)
        
        button_width = 10  # 统一按钮宽度
        button_padding = 3  # 按钮间距
        
        # 控制按钮 - 使用统一按钮样式
        # 开始任务按钮 - 启动设备选择对话框和计时器设置
        ttk.Button(control_frame, text="开始任务", 
                   width=button_width, 
                   command=self.start_task).pack(side=tk.LEFT, padx=button_padding)
        
        # 暂停所有按钮 - 暂停所有正在运行的设备线程           
        ttk.Button(control_frame, text="暂停所有", 
                   width=button_width, 
                   command=lambda: self.toggle_all_devices(True)).pack(side=tk.LEFT, padx=button_padding)
        
        # 继续所有按钮 - 继续所有暂停的设备线程           
        ttk.Button(control_frame, text="继续所有", 
                   width=button_width, 
                   command=lambda: self.toggle_all_devices(False)).pack(side=tk.LEFT, padx=button_padding)
        
        # 停止所有按钮 - 停止所有设备线程           
        ttk.Button(control_frame, text="停止所有", 
                   width=button_width, 
                   command=self.stop_all_devices).pack(side=tk.LEFT, padx=button_padding)
        
        # 刷新设备按钮 - 重新扫描并连接设备           
        ttk.Button(control_frame, text="刷新设备", 
                   width=button_width, 
                   command=lambda: self.refresh_devices(force=True)).pack(side=tk.LEFT, padx=button_padding)
        
        # 创建横向分隔线 - 视觉分隔
        separator = ttk.Separator(main_frame, orient="horizontal")
        separator.pack(fill=tk.X, pady=10)
        
        # 设备列表标题 - 区域标题
        device_title_frame = ttk.Frame(main_frame)
        device_title_frame.pack(fill=tk.X, pady=(0, 5))
        ttk.Label(device_title_frame, text="设备列表", font=self.normal_font).pack(side=tk.LEFT)
        
        # 设备列表区域 (可滚动) - 显示设备卡片
        device_container = ttk.Frame(main_frame)
        device_container.pack(fill=tk.BOTH, expand=True)
        
        # 创建Canvas和滚动条 - 支持多设备滚动显示
        canvas = tk.Canvas(device_container, background=self.frame_bg, highlightthickness=0)
        scrollbar = ttk.Scrollbar(device_container, orient="vertical", command=canvas.yview)
        
        # 创建内部框架放置设备卡片 - 实际容器
        self.devices_frame = ttk.Frame(canvas)
        
        # 配置滚动 - 确保Canvas根据内容调整滚动区域
        self.devices_frame.bind(
            "<Configure>",
            lambda e: canvas.configure(scrollregion=canvas.bbox("all"))
        )
        
        # 将设备框架添加到Canvas - 启用滚动功能
        canvas.create_window((0, 0), window=self.devices_frame, anchor="nw")
        canvas.configure(yscrollcommand=scrollbar.set)
        
        # 布局Canvas和滚动条
        canvas.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")
        
        # 日志区域分隔线 - 视觉分隔
        separator2 = ttk.Separator(main_frame, orient="horizontal")
        separator2.pack(fill=tk.X, pady=10)
        
        # 日志标题 - 区域标题
        log_title_frame = ttk.Frame(main_frame)
        log_title_frame.pack(fill=tk.X, pady=(0, 5))
        ttk.Label(log_title_frame, text="运行日志", font=self.normal_font).pack(side=tk.LEFT)
        
        # 日志区域 - 显示操作日志的文本区域
        log_frame = ttk.Frame(main_frame, style="Card.TFrame")
        log_frame.pack(fill=tk.BOTH, expand=True)
        
        # 创建可滚动文本区域显示日志
        self.log_text = scrolledtext.ScrolledText(log_frame, height=8, font=self.log_font)
        self.log_text.pack(fill=tk.BOTH, expand=True, padx=1, pady=1)
        
        # 日志颜色样式 - 不同类型日志使用不同颜色
        self.log_text.tag_configure("info", foreground="#333333")     # 普通信息 - 深灰色
        self.log_text.tag_configure("warning", foreground="#FF8C00")  # 警告信息 - 橙色
        self.log_text.tag_configure("error", foreground="#E81123")    # 错误信息 - 红色
        
        # 状态栏 - 显示整体状态信息
        status_frame = ttk.Frame(self.root, style="CardHeader.TFrame")
        status_frame.pack(side=tk.BOTTOM, fill=tk.X)
        
        # 状态信息标签
        self.status_bar = ttk.Label(status_frame, font=self.log_font, padding=(8, 3))
        self.status_bar.pack(side=tk.LEFT)
    
    def status_callback(self, log_entry):
        """
        状态回调函数
        
        当有新的日志记录时被调用，将更新UI的工作交给主线程的消息队列处理，
        避免直接从后台线程修改UI导致的线程安全问题。
        
        参数:
            log_entry: 日志条目字典，包含时间、设备、消息和级别信息
        """
        # 将状态更新工作交给主线程的消息队列处理
        pass
    
    def update_status(self):
        """
        更新状态显示
        
        更新日志文本区域和状态栏，显示最新的日志信息和设备统计数据。
        该方法应该只在主线程中调用，以确保UI更新的线程安全。
        """
        # 更新日志文本 - 显示最近的50条日志
        self.log_text.delete(1.0, tk.END)
        for entry in status_logger.get_history(50):
            level_tag = entry["level"]  # 使用日志级别作为标签名
            self.log_text.insert(tk.END, f"[{entry['time']}] [{entry['device']}] {entry['message']}\n", level_tag)
        self.log_text.see(tk.END)  # 自动滚动到最新日志
        
        # 更新状态栏 - 显示设备统计信息
        running_count = sum(1 for e in self.stop_events.values() if not e.is_set())
        paused_count = sum(1 for e in self.pause_events.values() if e.is_set())
        self.status_bar.config(text=f"运行中: {running_count} | 已暂停: {paused_count} | 总设备: {len(self.device_threads)}")
    
    def update_timer(self):
        """
        定时器更新函数
        
        每秒更新一次设备状态信息，包括运行时间、剩余时间和进度条。
        这是一个周期性执行的函数，由Tkinter的after机制调度。
        """
        try:
            # 更新设备状态 - 遍历所有设备线程
            for device_id, thread in self.device_threads.items():
                if thread.is_alive():
                    device_frame = self.device_frames.get(device_id)
                    if device_frame and hasattr(thread, "start_time"):
                        # 计算已运行时间（分钟）
                        elapsed = (time.time() - thread.start_time) / 60
                        # 计算剩余时间（分钟）
                        remaining = thread.run_duration - elapsed if hasattr(thread, "run_duration") else 0
                        
                        # 更新进度条 - 显示任务完成百分比
                        if remaining > 0 and thread.run_duration > 0:
                            progress = min(100, max(0, (elapsed / thread.run_duration) * 100))
                            device_frame["progress"].config(value=progress)
                            
                        # 更新时间标签 - 显示状态和时间信息
                        status_text = f"运行中" if not self.pause_events[device_id].is_set() else "已暂停"
                        device_frame["status"].config(text=f"状态: {status_text} | 已运行: {elapsed:.1f}分钟 | 剩余: {remaining:.1f}分钟")
        except Exception as e:
            logger.error(f"更新定时器错误: {str(e)}")
            
        # 每秒更新一次 - 保持UI响应性
        self.root.after(1000, self.update_timer)
    
    def create_device_card(self, parent_frame, device_id, device_info, width=340):
        """
        创建设备卡片 - 使用pack布局
        
        为每个设备创建一个卡片式界面，包含设备信息、状态、进度条和控制按钮。
        卡片使用固定宽度，确保界面整齐一致。
        
        参数:
            parent_frame: 父容器框架
            device_id: 设备唯一标识符
            device_info: 设备描述信息
            width: 卡片宽度，默认340像素
            
        返回:
            包含卡片UI组件引用的字典，用于后续更新
        """
        # 创建设备框架 - 固定大小的卡片
        frame = ttk.Frame(parent_frame, style="Card.TFrame", width=width, height=180)
        frame.pack(side=tk.LEFT, padx=10, pady=5)
        frame.pack_propagate(False)  # 防止内部组件影响Frame大小
        
        # 设备标题栏 - 显示设备名称
        header_frame = ttk.Frame(frame, style="CardHeader.TFrame")
        header_frame.pack(fill=tk.X)
        
        ttk.Label(header_frame, text=device_info, style="Header.TLabel").pack(side=tk.LEFT, padx=10, pady=5)
        
        # 内容区域 - 包含状态、进度和按钮
        content_frame = ttk.Frame(frame, style="Card.TFrame")
        content_frame.pack(fill=tk.BOTH, expand=True, padx=0)
        
        # 状态信息 - 显示运行状态和时间
        status_frame = ttk.Frame(content_frame, style="Card.TFrame")
        status_frame.pack(fill=tk.X, padx=10, pady=5)
        
        status_label = ttk.Label(status_frame, text="状态: 就绪", style="Card.TLabel")
        status_label.pack(side=tk.LEFT)
        
        # 进度条 - 显示任务完成进度
        progress_frame = ttk.Frame(content_frame, style="Card.TFrame")
        progress_frame.pack(fill=tk.X, padx=10, pady=(0, 5))
        
        progress = ttk.Progressbar(progress_frame, length=100, mode="determinate")
        progress.pack(fill=tk.X)
        
        # 按钮区域 - 设备控制按钮
        button_frame = ttk.Frame(content_frame, style="Card.TFrame")
        button_frame.pack(fill=tk.X, padx=10, pady=(0, 10))
        
        # 创建均匀的按钮，使用更紧凑的布局
        button_width = 7  # 按钮宽度
        pad = 2           # 按钮间距
        
        # 开始按钮 - 启动设备线程
        start_button = ttk.Button(button_frame, text="开始", width=button_width,
                  command=lambda d=device_id: self.start_device(d))
        start_button.pack(side=tk.LEFT, padx=pad)
        
        # 暂停/继续按钮 - 切换设备线程的暂停状态
        toggle_button = ttk.Button(button_frame, text="暂停/继续", width=button_width+2,
                  command=lambda d=device_id: self.toggle_device(d))
        toggle_button.pack(side=tk.LEFT, padx=pad)
        
        # 停止按钮 - 终止设备线程
        stop_button = ttk.Button(button_frame, text="停止", width=button_width,
                  command=lambda d=device_id: self.stop_device(d))
        stop_button.pack(side=tk.LEFT, padx=pad)
        
        # 截图按钮 - 获取设备当前屏幕截图
        screenshot_button = ttk.Button(button_frame, text="截图", width=button_width,
                  command=lambda d=device_id: self.take_screenshot(d))
        screenshot_button.pack(side=tk.LEFT, padx=pad)
        
        # 返回组件引用字典，用于后续更新
        return {
            "frame": frame,
            "status": status_label,
            "progress": progress,
            "start_button": start_button,
            "toggle_button": toggle_button,
            "stop_button": stop_button,
            "screenshot_button": screenshot_button
        }
    
    def refresh_devices(self, force=False):
        """刷新设备列表
        
        Args:
            force: 是否强制刷新，如果为False则优先使用缓存
        """
        # 如果已经在刷新中，则跳过
        if self.is_refreshing:
            return
            
        # 设置刷新标志
        self.is_refreshing = True
        
        try:
            status_logger.log("系统", "正在刷新设备列表...", "info")
            
            # 清空设备框架
            for widget in self.devices_frame.winfo_children():
                widget.destroy()
                
            # 存储设备框架
            self.device_frames = {}
            
            # 如果需要强制刷新设备，则清空现有设备列表
            if force and hasattr(self.auto_clicker, 'devices'):
                self.auto_clicker.devices = []
            
            # 获取设备列表
            devices = self.auto_clicker.list_devices()
            
            # 确保devices不为None
            if devices is None:
                devices = []
            
            if not devices:
                ttk.Label(self.devices_frame, text="未找到任何设备，请确保模拟器已启动").pack(pady=10)
                status_logger.log("系统", "未找到任何设备", "warning")
            else:
                # 完全使用pack布局，创建流式布局
                # 创建水平容器框架
                device_container = ttk.Frame(self.devices_frame)
                device_container.pack(fill=tk.BOTH, expand=True)
                
                # 计算每行可容纳的设备数量
                device_width = 340  # 设备卡片宽度
                padding = 10        # 设备间的内边距
                window_width = self.root.winfo_width() - 40  # 减去主框架边距
                columns = max(1, window_width // (device_width + 2*padding))
                
                # 创建设备卡片
                current_row = None
                current_col = 0
                
                for i, (device_id, device_info) in enumerate(devices):
                    # 确定当前设备应该放在哪一行
                    row_num = i // columns
                    
                    # 如果需要创建新行
                    if i % columns == 0:
                        current_row = ttk.Frame(device_container)
                        current_row.pack(fill=tk.X, pady=5)
                        current_col = 0
                    
                    # 创建设备卡片
                    device_frame = self.create_device_card(current_row, device_id, device_info, device_width)
                    self.device_frames[device_id] = device_frame
                    
                    # 下一列
                    current_col += 1
                
            status_logger.log("系统", f"设备刷新完成，共发现 {len(devices)} 个设备", "info")
            
            # 添加窗口大小变化事件处理，以便在调整窗口大小时重新布局
            self.root.bind("<Configure>", self.on_window_resize)
            
            # 保存当前窗口宽度
            self.last_width = self.root.winfo_width()
        finally:
            # 清除刷新标志
            self.is_refreshing = False
    
    def start_task(self):
        """开始任务"""
        # 获取运行时间
        duration = self.get_duration()
        if duration <= 0:
            return
            
        # 选择设备
        selected_devices = self.select_devices()
        if not selected_devices:
            return
            
        # 开始所有选择的设备
        for device_id, device_info in selected_devices:
            self.start_device(device_id, device_info, duration)
    
    def get_duration(self):
        """获取运行时间"""
        dialog = DurationDialog(self.root)
        self.root.wait_window(dialog.top)
        return dialog.duration
    
    def select_devices(self):
        """选择设备"""
        devices = self.auto_clicker.devices
        if not devices:
            messagebox.showinfo("提示", "未找到任何设备，请先刷新设备列表")
            return []
            
        dialog = DeviceSelectionDialog(self.root, devices)
        self.root.wait_window(dialog.top)
        return dialog.selected_devices
    
    def start_device(self, device_id, device_info=None, duration=None):
        """开始设备运行"""
        if not device_info:
            for d_id, d_info in self.auto_clicker.devices:
                if d_id == device_id:
                    device_info = d_info
                    break
        
        if not duration:
            duration = self.get_duration()
            if duration <= 0:
                return
                
        # 如果设备已经在运行，先停止
        self.stop_device(device_id)
        
        # 创建停止事件
        stop_event = threading.Event()
        self.stop_events[device_id] = stop_event
        
        # 创建暂停事件
        pause_event = threading.Event()
        self.pause_events[device_id] = pause_event
        
        # 创建并启动线程
        thread = threading.Thread(
            target=self.device_thread_func,
            args=(device_id, device_info, stop_event, pause_event, duration),
            name=f"Device-{device_id}"
        )
        thread.daemon = True
        thread.start_time = time.time()
        thread.run_duration = duration
        
        # 先将线程添加到字典中
        self.device_threads[device_id] = thread
        
        # 启动线程
        try:
            thread.start()
            status_logger.log(device_info, f"开始运行，计划运行时间: {duration}分钟", "info")
        except Exception as e:
            status_logger.log(device_info, f"启动线程失败: {str(e)}", "error")
            # 如果启动失败，从字典中移除
            if device_id in self.device_threads:
                del self.device_threads[device_id]
            if device_id in self.stop_events:
                del self.stop_events[device_id]
            if device_id in self.pause_events:
                del self.pause_events[device_id]
        
        # 更新UI状态
        self.update_status()
    
    def device_thread_func(self, device_id, device_info, stop_event, pause_event, duration):
        """设备线程函数"""
        try:
            status_logger.log(device_info, "线程启动", "info")
            
            end_time = time.time() + (duration * 60)  # 转换为秒
            last_log_time = 0  # 上次详细日志时间
            
            # 按钮点击记录和冷却时间
            last_click_time = 0
            click_cooldown = 1.0  # 按钮点击冷却时间(秒)
            
            # 实际执行点击操作
            while not stop_event.is_set() and time.time() < end_time:
                if pause_event.is_set():
                    time.sleep(1)
                    continue
                
                try:
                    current_time = time.time()
                    # 控制日志输出频率，每10秒输出一次详细日志
                    verbose_log = (current_time - last_log_time) > 10
                    
                    # 获取屏幕截图
                    screen = self.auto_clicker.get_screen(device_id)
                    if screen is None:
                        status_logger.log(device_info, "获取屏幕截图失败", "error")
                        time.sleep(1)
                        continue
                    
                    # 检查失败状态
                    lose_pos = self.auto_clicker.find_template(device_id, "lose", screen)
                    if lose_pos:
                        status_logger.log(device_info, "检测到失败", "warning")
                        pause_event.set()  # 暂停执行
                        # 通知UI更新状态
                        message_queue.put(("log_event", {"time": time.strftime("%H:%M:%S"), 
                                                        "device": device_info, 
                                                        "message": "检测到失败，已暂停", 
                                                        "level": "warning"}))
                        # 立即更新UI状态
                        self.update_status()
                        break
                    
                    # 检查突破券不足状态
                    notupo_pos = self.auto_clicker.find_template(device_id, "notupo", screen)
                    if notupo_pos:
                        status_logger.log(device_info, "检测到突破券不足", "warning")
                        pause_event.set()  # 暂停执行
                        # 通知UI更新状态
                        message_queue.put(("log_event", {"time": time.strftime("%H:%M:%S"), 
                                                        "device": device_info, 
                                                        "message": "检测到突破券不足，已暂停", 
                                                        "level": "warning"}))
                        # 立即更新UI状态
                        self.update_status()
                        break
                    
                    # 检查是否可以点击（点击冷却时间检查）
                    if current_time - last_click_time < click_cooldown:
                        # 如果在冷却中，短暂等待
                        time.sleep(0.1)
                        continue
                    
                    # 检查可点击的按钮
                    clicked = False
                    for template_name in self.auto_clicker.get_template_names():
                        if stop_event.is_set() or pause_event.is_set():
                            break
                            
                        pos = self.auto_clicker.find_template(device_id, template_name, screen)
                        if pos:
                            x, y = pos
                            if verbose_log:
                                status_logger.log(device_info, f"找到按钮 {template_name}", "info")
                                last_log_time = current_time
                            
                            # 特殊处理button10 (确定按钮)
                            if template_name == "button10":
                                if self.auto_clicker.random_click(device_id, x, y, template_name):
                                    status_logger.log(device_info, f"点击了 button10", "info")
                                    clicked = True
                                    last_click_time = current_time
                                    # 点击后立即再次检查notupo
                                    time.sleep(0.5)
                                    check_screen = self.auto_clicker.get_screen(device_id)
                                    if check_screen is not None:
                                        if self.auto_clicker.find_template(device_id, "notupo", check_screen):
                                            status_logger.log(device_info, "点击button10后检测到突破券不足", "warning")
                                            pause_event.set()  # 暂停执行
                                            # 通知UI更新状态
                                            message_queue.put(("log_event", {"time": time.strftime("%H:%M:%S"), 
                                                                            "device": device_info, 
                                                                            "message": "检测到突破券不足，已暂停", 
                                                                            "level": "warning"}))
                                            # 立即更新UI状态
                                            self.update_status()
                                            break
                                    break
                            
                            # 特殊处理button7 (挑战按钮)
                            elif template_name == "button7":
                                wait_time = random.uniform(8, 10)  # 与auto_click.py中保持一致
                                status_logger.log(device_info, f"检测到button7，等待 {wait_time:.1f} 秒后点击", "info")
                                time.sleep(wait_time)
                                if self.auto_clicker.random_click(device_id, x, y, template_name):
                                    status_logger.log(device_info, f"点击了 button7", "info")
                                    clicked = True
                                    last_click_time = current_time
                                    self.auto_clicker.random_delay(template_name, 1, 3)
                                    break
                            
                            # 处理其他按钮
                            else:
                                if self.auto_clicker.random_click(device_id, x, y, template_name):
                                    status_logger.log(device_info, f"点击了 {template_name}", "info")
                                    clicked = True
                                    last_click_time = current_time
                                    self.auto_clicker.random_delay(template_name, 1, 3)
                                    break
                    
                    if not clicked:
                        # 如果未点击任何按钮
                        if verbose_log:
                            status_logger.log(device_info, "等待可点击目标...", "info")
                            last_log_time = current_time
                        # 等待一段时间
                        time.sleep(0.5)
                    
                except Exception as e:
                    status_logger.log(device_info, f"点击操作出错: {str(e)}", "error")
                    time.sleep(2)
            
            status_logger.log(device_info, "线程完成", "info")
        except Exception as e:
            status_logger.log(device_info, f"线程错误: {str(e)}", "error")
        finally:
            # 确保线程结束时清理资源
            status_logger.log(device_info, "线程清理资源", "info")
    
    def toggle_device(self, device_id):
        """暂停/继续设备"""
        if device_id in self.pause_events:
            pause_event = self.pause_events[device_id]
            if pause_event.is_set():
                pause_event.clear()
                status_logger.log(device_id, "继续运行", "info")
                
                # 如果线程已停止但未标记为停止，重新启动线程
                if device_id in self.device_threads:
                    thread = self.device_threads[device_id]
                    if not thread.is_alive() and device_id in self.stop_events and not self.stop_events[device_id].is_set():
                        # 获取设备信息
                        device_info = None
                        for d_id, d_info in self.auto_clicker.devices:
                            if d_id == device_id:
                                device_info = d_info
                                break
                                
                        if device_info and hasattr(thread, "run_duration"):
                            duration = thread.run_duration
                            # 重新启动线程
                            self.start_device(device_id, device_info, duration)
                            status_logger.log(device_info, f"重新启动线程", "info")
            else:
                pause_event.set()
                status_logger.log(device_id, "已暂停", "info")
                
            # 更新UI状态
            self.update_status()
    
    def toggle_all_devices(self, pause=True):
        """暂停/继续所有设备"""
        for device_id in self.pause_events:
            if pause:
                self.pause_events[device_id].set()
            else:
                self.pause_events[device_id].clear()
                
        status_logger.log("系统", f"{'已暂停' if pause else '已继续'}所有设备", "info")
        
        # 更新UI状态
        self.update_status()
    
    def stop_device(self, device_id):
        """停止设备"""
        if device_id in self.stop_events:
            self.stop_events[device_id].set()
            if device_id in self.device_threads:
                thread = self.device_threads[device_id]
                if thread.is_alive():  # 检查线程是否已启动
                    thread.join(0.1)  # 等待线程结束
                status_logger.log(device_id, "已停止", "info")
                
                # 重置进度条
                if device_id in self.device_frames:
                    self.device_frames[device_id]["progress"].config(value=0)
                    # 更新状态标签
                    self.device_frames[device_id]["status"].config(text="状态: 已停止")
                
                # 更新UI状态
                self.update_status()
    
    def stop_all_devices(self):
        """停止所有设备"""
        # 设置所有停止事件
        for device_id in list(self.stop_events.keys()):
            self.stop_events[device_id].set()
            
            # 重置进度条
            if device_id in self.device_frames:
                self.device_frames[device_id]["progress"].config(value=0)
                self.device_frames[device_id]["status"].config(text="状态: 已停止")
            
        # 等待所有线程结束
        for device_id, thread in list(self.device_threads.items()):
            if thread.is_alive():  # 检查线程是否已启动
                thread.join(0.1)
            
        status_logger.log("系统", "已停止所有设备", "info")
        
        # 更新UI状态
        self.update_status()
    
    def take_screenshot(self, device_id):
        """获取设备截图"""
        try:
            # 创建临时文件夹
            os.makedirs("temp", exist_ok=True)
            
            # 执行截图命令
            try:
                # 截图保存到设备
                cmd = f'"{self.auto_clicker.adb_path}" -s {device_id} shell screencap -p /sdcard/screenshot.png'
                subprocess.check_output(cmd, shell=True)
                
                # 下载到本地
                screenshot_path = f"temp/screenshot_{device_id.replace(':', '_')}.png"
                cmd = f'"{self.auto_clicker.adb_path}" -s {device_id} pull /sdcard/screenshot.png {screenshot_path}'
                subprocess.check_output(cmd, shell=True)
                
                # 删除设备上的截图
                cmd = f'"{self.auto_clicker.adb_path}" -s {device_id} shell rm /sdcard/screenshot.png'
                subprocess.check_output(cmd, shell=True)
                
                status_logger.log(device_id, f"获取屏幕截图成功: {screenshot_path}", "info")
                
                # 打开截图
                import webbrowser
                webbrowser.open(screenshot_path)
            except Exception as e:
                status_logger.log(device_id, "获取屏幕截图失败", "error")
                messagebox.showerror("错误", f"获取设备 {device_id} 的屏幕截图失败: {str(e)}")
        except Exception as e:
            status_logger.log(device_id, f"截图错误: {str(e)}", "error")
            messagebox.showerror("错误", f"截图错误: {str(e)}")
    
    def on_closing(self):
        """窗口关闭处理"""
        try:
            # 停止所有设备
            self.stop_all_devices()
            # 等待一点时间确保所有线程都有机会清理
            time.sleep(0.2)
            # 保存配置
            self.save_config()
            # 关闭窗口
            self.root.destroy()
        except Exception as e:
            # 如果出现异常，记录错误并强制关闭
            logger.error(f"关闭窗口时出错: {str(e)}")
            self.root.destroy()

class DurationDialog:
    """运行时间对话框 - Windows游戏风格"""
    def __init__(self, parent):
        self.duration = 0
        self.parent = parent
        
        self.top = tk.Toplevel(parent)
        self.top.title("设置运行时间")
        self.top.geometry("380x320")  # 调整对话框尺寸
        self.top.resizable(False, False)  # 禁止调整大小
        self.top.transient(parent)
        self.top.grab_set()
        
        # 设置对话框居中
        self.center_window()
        
        # 应用Windows游戏风格 
        if hasattr(parent, 'normal_font'):
            self.top.option_add("*Font", parent.normal_font)
            title_font = parent.title_font
            button_font = parent.button_font
            # 获取父窗口样式和颜色
            self.style = parent.style
            self.bg_color = parent.bg_color
            self.accent_color = parent.accent_color if hasattr(parent, 'accent_color') else "#007ACC"
            self.frame_bg = parent.frame_bg if hasattr(parent, 'frame_bg') else "#FFFFFF"
        else:
            title_font = font.Font(family="微软雅黑", size=12, weight="bold")
            button_font = font.Font(family="微软雅黑", size=10, weight="bold")
            # 创建样式
            self.style = ttk.Style()
            self.bg_color = "#EFEFEF"  # 默认背景色
            self.accent_color = "#007ACC"  # 默认强调色
            self.frame_bg = "#FFFFFF"  # 默认框架背景色
            
        # 设置背景色
        self.top.configure(bg=self.bg_color)
        
        # 内容框架
        main_frame = ttk.Frame(self.top)
        main_frame.pack(fill=tk.BOTH, expand=True, padx=15, pady=15)
        
        # 标题
        title_frame = ttk.Frame(main_frame)
        title_frame.pack(fill=tk.X, pady=(0, 10))
        ttk.Label(title_frame, text="请设置运行时间", font=title_font).pack(pady=5)
        
        # 输入框框架 - 使用卡片风格
        entry_frame = ttk.Frame(main_frame, style="Card.TFrame")
        entry_frame.pack(fill=tk.X, pady=10)
        
        # 输入框标签和输入区域
        input_container = ttk.Frame(entry_frame, style="Card.TFrame")
        input_container.pack(fill=tk.X, padx=15, pady=15)
        
        ttk.Label(input_container, text="运行时间(分钟):", style="Card.TLabel").pack(side=tk.LEFT, padx=(0, 10))
        
        # 创建样式化的输入框
        self.duration_var = tk.StringVar(value="60")
        entry = ttk.Entry(input_container, textvariable=self.duration_var, width=8, font=button_font, justify='center')
        entry.pack(side=tk.LEFT, padx=5)
        entry.focus_set()
        
        # 快速选择区域
        quick_frame = ttk.Frame(entry_frame, style="Card.TFrame")
        quick_frame.pack(fill=tk.X, padx=15, pady=(0, 15))
        
        ttk.Label(quick_frame, text="快速选择:", style="Card.TLabel").pack(anchor=tk.W, pady=(0, 5))
        
        # 快速选择按钮区域
        btn_container = ttk.Frame(quick_frame, style="Card.TFrame")
        btn_container.pack(fill=tk.X)
        
        # 配置按钮容器的列权重
        for i in range(3):
            btn_container.columnconfigure(i, weight=1)
        
        # 创建快速选择按钮
        durations = [(30, "30分钟"), (60, "1小时"), (120, "2小时")]
        for i, (duration, text) in enumerate(durations):
            btn = ttk.Button(
                btn_container, 
                text=text, 
                command=lambda d=duration: self.duration_var.set(str(d))
            )
            btn.grid(row=0, column=i, padx=5, pady=5, sticky="ew")
        
        # 底部按钮区域
        button_frame = ttk.Frame(main_frame)
        button_frame.pack(fill=tk.X, pady=(15, 0))
        
        # 设置按钮区域居右对齐
        button_frame.columnconfigure(0, weight=1)  # 左侧空白区域拉伸
        
        # 确定和取消按钮
        cancel_btn = ttk.Button(button_frame, text="取消", command=self.cancel, width=10)
        cancel_btn.grid(row=0, column=1, padx=(0, 5))
        
        ok_btn = ttk.Button(button_frame, text="确定", command=self.ok, width=10)
        ok_btn.grid(row=0, column=2, padx=(5, 0))
        
        # 绑定回车键和Escape键
        self.top.bind("<Return>", lambda event: self.ok())
        self.top.bind("<Escape>", lambda event: self.cancel())
        
    def center_window(self):
        """居中对话框"""
        self.top.update_idletasks()
        width = self.top.winfo_width()
        height = self.top.winfo_height()
        x = (self.parent.winfo_width() // 2) - (width // 2) + self.parent.winfo_x()
        y = (self.parent.winfo_height() // 2) - (height // 2) + self.parent.winfo_y()
        self.top.geometry(f"{width}x{height}+{x}+{y}")
        
    def ok(self):
        """确定按钮处理"""
        try:
            self.duration = int(self.duration_var.get())
            if self.duration <= 0:
                raise ValueError("运行时间必须大于0")
                
            self.top.destroy()
        except ValueError as e:
            messagebox.showerror("错误", f"无效的运行时间: {str(e)}")
    
    def cancel(self):
        """取消按钮处理"""
        self.duration = 0
        self.top.destroy()

class DeviceSelectionDialog:
    """设备选择对话框 - Windows游戏风格"""
    def __init__(self, parent, devices):
        self.devices = devices
        self.selected_devices = []
        self.parent = parent
        
        self.top = tk.Toplevel(parent)
        self.top.title("选择设备")
        self.top.geometry("400x450")  # 增加高度确保按钮可见
        self.top.minsize(400, 450)  # 设置最小尺寸
        self.top.resizable(False, False)  # 禁止调整大小
        self.top.transient(parent)
        self.top.grab_set()
        
        # 设置对话框居中
        self.center_window()
        
        # 应用Windows游戏风格
        if hasattr(parent, 'normal_font'):
            self.top.option_add("*Font", parent.normal_font)
            title_font = parent.title_font
            # 获取父窗口样式和颜色
            self.style = parent.style
            self.bg_color = parent.bg_color
            # 获取frame_bg和header_bg
            self.frame_bg = parent.frame_bg if hasattr(parent, 'frame_bg') else "#FFFFFF"
            self.header_bg = parent.header_bg if hasattr(parent, 'header_bg') else "#F0F0F0"
        else:
            title_font = font.Font(family="微软雅黑", size=12, weight="bold")
            # 创建样式
            self.style = ttk.Style()
            self.bg_color = "#EFEFEF"  # 默认背景色
            self.frame_bg = "#FFFFFF"  # 默认框架背景色
            self.header_bg = "#F0F0F0" # 默认标题栏背景色
        
        # 设置背景色
        self.top.configure(bg=self.bg_color)
        
        # 使用grid布局管理器
        self.top.grid_columnconfigure(0, weight=1)
        self.top.grid_rowconfigure(1, weight=1)  # 设备列表区域可扩展
        
        # 标题
        title_frame = ttk.Frame(self.top, style="TFrame")
        title_frame.grid(row=0, column=0, padx=15, pady=(15,5), sticky="ew")
        ttk.Label(title_frame, text="请选择要运行的设备", font=title_font).pack(pady=5)
        
        # 全选复选框 - 独立框架
        select_all_frame = ttk.Frame(self.top, style="CardHeader.TFrame")
        select_all_frame.grid(row=1, column=0, padx=15, pady=0, sticky="new")
        
        select_all_var = tk.BooleanVar(value=True)  # 默认全选
        select_all = tk.Checkbutton(select_all_frame, text="全选", 
                                  variable=select_all_var,
                                  command=lambda: self.select_all(select_all_var.get()),
                                  bg=self.header_bg if hasattr(self, 'header_bg') else "#F0F0F0",
                                  font=parent.normal_font if hasattr(parent, 'normal_font') else None)
        select_all.pack(anchor=tk.W, padx=10, pady=8)
        
        # 设备选择框 - 卡片式设计
        device_panel = ttk.Frame(self.top, style="Card.TFrame")
        device_panel.grid(row=2, column=0, padx=15, pady=(0,10), sticky="nsew")
        
        # 创建滚动区域
        canvas = tk.Canvas(device_panel, background=self.frame_bg, highlightthickness=0)
        scrollbar = ttk.Scrollbar(device_panel, orient="vertical", command=canvas.yview)
        
        # 内部框架
        listbox_frame = ttk.Frame(canvas, style="Card.TFrame")
        
        # 添加设备复选框
        self.device_vars = {}
        for i, (device_id, device_info) in enumerate(devices):
            var = tk.BooleanVar(value=True)  # 默认选中所有设备
            self.device_vars[device_id] = var
            
            # 创建交替的行颜色
            bg_color = "#F9F9F9" if i % 2 == 0 else self.frame_bg
            device_row = ttk.Frame(listbox_frame)
            device_row.pack(fill=tk.X, pady=1)
            
            # 使用tkinter原生复选框可以设置背景色
            cb = tk.Checkbutton(device_row, text=f"{device_info}", 
                               variable=var, bg=bg_color,
                               font=parent.normal_font if hasattr(parent, 'normal_font') else None)
            cb.pack(anchor=tk.W, padx=10, pady=6, fill=tk.X)
        
        # 配置Canvas
        canvas.create_window((0, 0), window=listbox_frame, anchor="nw", width=370)  # 固定宽度
        listbox_frame.bind("<Configure>", lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
        canvas.configure(yscrollcommand=scrollbar.set)
        
        canvas.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")
        
        # 按钮区域 - 使用固定位置
        btn_frame = ttk.Frame(self.top, style="TFrame")
        btn_frame.grid(row=3, column=0, padx=15, pady=(0,15), sticky="ew")
        
        # 让按钮区域的内容居右
        btn_frame.columnconfigure(0, weight=1)
        
        ttk.Button(btn_frame, text="取消", command=self.cancel, width=10).grid(row=0, column=1, padx=5)
        ttk.Button(btn_frame, text="确定", command=self.ok, width=10).grid(row=0, column=2, padx=5)

    def center_window(self):
        """居中对话框"""
        self.top.update_idletasks()
        width = self.top.winfo_width()
        height = self.top.winfo_height()
        x = (self.parent.winfo_width() // 2) - (width // 2) + self.parent.winfo_x()
        y = (self.parent.winfo_height() // 2) - (height // 2) + self.parent.winfo_y()
        self.top.geometry(f"{width}x{height}+{x}+{y}")
        
    def select_all(self, select):
        """全选/取消全选"""
        for var in self.device_vars.values():
            var.set(select)
    
    def ok(self):
        """确定按钮处理"""
        self.selected_devices = []
        
        for device_id, var in self.device_vars.items():
            if var.get():
                for d_id, d_info in self.devices:
                    if d_id == device_id:
                        self.selected_devices.append((d_id, d_info))
                        break
        
        if not self.selected_devices:
            messagebox.showwarning("警告", "请至少选择一个设备")
            return
            
        self.top.destroy()
        
    def cancel(self):
        """取消按钮处理"""
        self.selected_devices = []
        self.top.destroy()

class AutoClicker:
    """自动点击器类"""
    def __init__(self):
        self.adb_path = r"F:\testclinm\platform-tools\adb.exe"
        self.mumu_manager_path = r"F:\MuMu Player 12\shell\MuMuManager.exe"
        self.devices = []  # 存储设备列表[(device_id, device_info), ...]
        
        # 图像识别相关设置
        self.template_dir = r"F:\testclinm\templates"
        self.threshold = 0.8  # 图像匹配阈值
        
    def execute_adb_command(self, device_id, command):
        """执行ADB命令"""
        try:
            full_command = f'"{self.adb_path}" -s {device_id} {command}'
            result = subprocess.check_output(full_command, shell=True, text=True, encoding='utf-8', errors='ignore')
            return result.strip()
        except Exception as e:
            print(f"ADB命令执行错误: {str(e)}")
            return ""
    
    def take_screenshot(self, device_id):
        """获取屏幕截图"""
        try:
            # 创建临时文件夹
            os.makedirs("temp", exist_ok=True)
            
            # 截图保存到设备
            self.execute_adb_command(device_id, "shell screencap -p /sdcard/screenshot.png")
            
            # 下载到本地
            screenshot_path = f"temp/screenshot_{device_id.replace(':', '_')}.png"
            self.execute_adb_command(device_id, f"pull /sdcard/screenshot.png {screenshot_path}")
            
            # 删除设备上的截图
            self.execute_adb_command(device_id, "shell rm /sdcard/screenshot.png")
            
            return screenshot_path
        except Exception as e:
            print(f"截图错误: {str(e)}")
            return None
    
    def tap(self, device_id, x, y):
        """点击屏幕坐标"""
        try:
            self.execute_adb_command(device_id, f"shell input tap {x} {y}")
            return True
        except Exception as e:
            print(f"点击错误: {str(e)}")
            return False
    
    def find_image(self, screenshot_path, template_path):
        """查找图像"""
        try:
            import cv2
            import numpy as np
            
            # 读取图像
            screenshot = cv2.imread(screenshot_path)
            template = cv2.imread(template_path)
            
            if screenshot is None or template is None:
                return None
            
            # 执行模板匹配
            result = cv2.matchTemplate(screenshot, template, cv2.TM_CCOEFF_NORMED)
            min_val, max_val, min_loc, max_loc = cv2.minMaxLoc(result)
            
            if max_val >= self.threshold:
                # 计算中心坐标
                h, w = template.shape[:2]
                center_x = max_loc[0] + w // 2
                center_y = max_loc[1] + h // 2
                return (center_x, center_y, max_val)
            
            return None
        except Exception as e:
            print(f"图像识别错误: {str(e)}")
            return None
    
    def find_template(self, device_id, template_name, screen):
        """查找模板图片位置，接收已有截图"""
        try:
            # 读取模板图片
            template_path = os.path.join(self.template_dir, f"{template_name}.png")
            if not os.path.exists(template_path):
                print(f"无法找到模板图片文件: {template_path}")
                return None
                
            import cv2
            import numpy as np
            
            template = cv2.imread(template_path)
            if template is None:
                print(f"无法读取模板图片: {template_name}")
                return None
                
            # 模板匹配
            result = cv2.matchTemplate(screen, template, cv2.TM_CCOEFF_NORMED)
            min_val, max_val, min_loc, max_loc = cv2.minMaxLoc(result)
            
            if max_val >= self.threshold:
                # 计算中心坐标
                h, w = template.shape[:2]
                center_x = max_loc[0] + w // 2
                center_y = max_loc[1] + h // 2
                
                print(f"找到图片 {template_name}, 匹配度: {max_val:.2f}")
                return (center_x, center_y)
            
            return None
        except Exception as e:
            print(f"查找模板图片出错: {str(e)}")
            return None
    
    def get_screen(self, device_id):
        """获取屏幕截图"""
        try:
            # 截图并保存到临时文件
            screenshot_path = self.take_screenshot(device_id)
            if not screenshot_path:
                return None
                
            # 读取截图
            import cv2
            screen = cv2.imread(screenshot_path)
            
            # 删除临时文件
            try:
                os.remove(screenshot_path)
            except:
                pass
                
            return screen
        except Exception as e:
            print(f"获取屏幕截图出错: {str(e)}")
            return None
    
    def random_click(self, device_id, x, y, template_name=None):
        """随机偏移点击"""
        try:
            # 计算随机偏移
            click_min = 5  # 最小偏移量
            click_max = 30  # 最大偏移量
            import numpy as np
            
            offset = random.randint(click_min, click_max)
            angle = random.uniform(0, 2 * np.pi)
            offset_x = int(offset * np.cos(angle))
            offset_y = int(offset * np.sin(angle))
            
            # 计算偏移后的点击坐标
            click_x = x + offset_x
            click_y = y + offset_y
            
            # 执行点击
            print(f"点击坐标: ({click_x}, {click_y}), 偏移量: {offset}")
            self.tap(device_id, click_x, click_y)
            return True
        except Exception as e:
            print(f"点击操作失败: {str(e)}")
            return False
    
    def random_delay(self, template_name, min_delay=1, max_delay=3):
        """随机延迟"""
        delay = random.uniform(min_delay, max_delay)
        print(f"等待 {delay:.2f} 秒")
        time.sleep(delay)
        
    def get_template_names(self):
        """获取所有模板名称（不含扩展名）"""
        template_names = []
        try:
            for file in os.listdir(self.template_dir):
                if file.endswith('.png'):
                    template_names.append(os.path.splitext(file)[0])
        except Exception as e:
            print(f"获取模板名称错误: {str(e)}")
        
        return template_names
    
    def get_template_paths(self):
        """获取所有模板图像路径"""
        template_paths = []
        try:
            for root, dirs, files in os.walk(self.template_dir):
                for file in files:
                    if file.endswith(('.png', '.jpg', '.jpeg')):
                        template_paths.append(os.path.join(root, file))
        except Exception as e:
            print(f"获取模板路径错误: {str(e)}")
        
        return template_paths
        
    def detect_and_click(self, device_id, template_name):
        """检测图像并点击"""
        try:
            # 获取截图
            screenshot_path = self.take_screenshot(device_id)
            if not screenshot_path:
                return False, "截图失败"
            
            # 查找模板路径
            template_path = None
            for path in self.get_template_paths():
                if template_name in os.path.basename(path):
                    template_path = path
                    break
            
            if not template_path:
                return False, f"未找到模板: {template_name}"
            
            # 查找图像
            result = self.find_image(screenshot_path, template_path)
            if result:
                x, y, confidence = result
                # 点击
                self.tap(device_id, x, y)
                return True, f"找到 {template_name} 并点击 ({x}, {y}), 置信度: {confidence:.2f}"
            
            return False, f"未在屏幕上找到: {template_name}"
        except Exception as e:
            return False, f"检测和点击错误: {str(e)}"
    
    def list_devices(self) -> List[Tuple[str, str]]:
        """
        获取连接的设备列表
        返回格式：[(device_id, device_info), ...]
        """
        # 如果已有设备列表且不为空，直接返回
        if hasattr(self, 'devices') and self.devices:
            status_logger.log("系统", f"使用缓存的设备列表，共 {len(self.devices)} 个设备", "info")
            return self.devices
            
        devices = []
        
        try:
            print("\n正在搜索设备...")
            status_logger.log("系统", "正在刷新设备列表...", "info")
            
            # 重启ADB服务器，只在第一次搜索时执行
            if not hasattr(self, '_adb_initialized'):
                print("初始化ADB服务器...")
                os.system(f'"{self.adb_path}" kill-server')
                time.sleep(1)
                os.system(f'"{self.adb_path}" start-server')
                time.sleep(2)
                self._adb_initialized = True
            
            # 尝试使用MuMuManager获取模拟器信息
            print("\n尝试使用MuMuManager获取模拟器信息...")
            try:
                cmd = f'"{self.mumu_manager_path}" info -v all'
                print(f"执行命令: {cmd}")
                mumu_info = subprocess.check_output(cmd, shell=True, text=True, encoding='utf-8', errors='ignore')
                print("MuMuManager输出:")
                print(mumu_info)
                
                # 解析MuMu模拟器信息
                mumu_devices = self.parse_mumu_info(mumu_info)
                if mumu_devices:
                    for device_id, device_info in mumu_devices:
                        print(f"通过MuMuManager找到设备: {device_info} ({device_id})")
                        devices.append((device_id, device_info))
            except Exception as e:
                print(f"获取MuMu模拟器信息失败: {e}")
                traceback.print_exc()
            
            # 如果MuMuManager方法失败，尝试常规ADB方法
            if not devices:
                print("\n使用常规ADB方法连接模拟器...")
                # 尝试连接MuMu模拟器的常见端口
                print("\n尝试连接MuMu模拟器常见端口...")
                # 首先是MuMu模拟器12的主要端口
                mumu_ports = [16384, 16416, 16448, 16480, 16512, 16544]
                # 其他常见端口
                other_ports = [21503, 62001, 59000, 5555, 5556, 5557, 5558, 5559, 7555]
                
                all_ports = mumu_ports + other_ports
                
                for port in all_ports:
                    device_id = f"127.0.0.1:{port}"
                    print(f"尝试连接: {device_id}")
                    
                    # 使用os.system避免编码问题
                    os.system(f'"{self.adb_path}" connect {device_id}')
                    # 简短暂停，让连接建立
                    time.sleep(0.5)
                
                # 获取当前连接的设备列表
                print("\n获取当前已连接的设备列表...")
                # 使用os.popen，并显式设置编码
                try:
                    cmd = f'"{self.adb_path}" devices'
                    devices_output = subprocess.check_output(cmd, shell=True, text=True, encoding='utf-8', errors='ignore')
                    print(devices_output)
                except Exception as e:
                    print(f"获取设备列表出错: {e}")
                    # 回退方法
                    devices_output = os.popen(cmd).read()
                    print(devices_output)
                
                # 解析设备列表
                lines = devices_output.strip().split('\n')
                if len(lines) > 1:  # 至少有标题行
                    for line in lines[1:]:  # 跳过标题行 "List of devices attached"
                        parts = line.strip().split()
                        if len(parts) >= 2 and parts[1] == 'device':
                            device_id = parts[0]
                            print(f"\n检查设备: {device_id}")
                            
                            # 尝试获取设备信息，使用subprocess避免编码问题
                            try:
                                # 获取设备型号
                                model_cmd = f'"{self.adb_path}" -s {device_id} shell getprop ro.product.model'
                                model = subprocess.check_output(model_cmd, shell=True, text=True, encoding='utf-8', errors='ignore').strip()
                                
                                # 获取设备品牌
                                brand_cmd = f'"{self.adb_path}" -s {device_id} shell getprop ro.product.brand'
                                brand = subprocess.check_output(brand_cmd, shell=True, text=True, encoding='utf-8', errors='ignore').strip()
                            except Exception:
                                # 如果subprocess失败，使用os.popen
                                model = os.popen(model_cmd).read().strip()
                                brand = os.popen(brand_cmd).read().strip()
                            
                            # 检查是否为MuMu实例并添加标识
                            device_info = f"{brand} {model}"
                            if device_id.startswith("127.0.0.1:"):
                                port = int(device_id.split(":")[1])
                                if port == 16384:
                                    device_info += " [MuMu主模拟器]"
                                elif port in mumu_ports:
                                    index = mumu_ports.index(port)
                                    device_info += f" [MuMu模拟器-{index}]"
                                else:
                                    device_info += f" [MuMu模拟器(端口:{port})]"
                            
                            print(f"  设备信息: {device_info}")
                            devices.append((device_id, device_info))
            
            # 显示最终结果
            if not devices:
                print("\n未找到任何目标设备")
                print("\n请检查:")
                print("1. 模拟器是否已启动")
                print("2. 模拟器设置中的ADB调试是否已开启")
            else:
                print(f"\n共找到 {len(devices)} 个设备:")
                for i, (device_id, device_info) in enumerate(devices, 1):
                    print(f"  {i}. {device_info} ({device_id})")
                
                # 确保设备列表中没有重复的设备
                unique_devices = []
                device_ids = set()
                
                for device_id, device_info in devices:
                    if device_id not in device_ids:
                        device_ids.add(device_id)
                        unique_devices.append((device_id, device_info))
                
                if len(unique_devices) != len(devices):
                    print(f"注意: 去除了 {len(devices) - len(unique_devices)} 个重复设备")
                
                # 更新设备列表
                self.devices = unique_devices
                status_logger.log("系统", f"找到 {len(unique_devices)} 个设备", "info")
                
                # 使用唯一设备列表作为返回值
                devices = unique_devices
            
            # 确保返回的是列表而不是None
            return devices
            
        except Exception as e:
            print(f"列出设备出错: {str(e)}")
            traceback.print_exc()
            # 返回空列表而不是None
            return []
    
    def parse_mumu_info(self, info_text):
        """解析MuMuManager输出的信息"""
        devices = []
        try:
            # 尝试解析为JSON
            import json
            try:
                data = json.loads(info_text)
                print(f"成功解析MuMu信息为JSON，包含 {len(data)} 个模拟器配置")
                
                # 遍历每个模拟器实例
                for instance_id, instance_info in data.items():
                    # 检查实例是否已启动
                    if instance_info.get("is_process_started") and instance_info.get("adb_port"):
                        name = instance_info.get("name", f"MuMu模拟器 {instance_id}")
                        port = instance_info.get("adb_port")
                        
                        device_id = f"127.0.0.1:{port}"
                        device_info = name
                        
                        # 尝试连接设备
                        print(f"尝试连接MuMu设备: {device_id}")
                        os.system(f'"{self.adb_path}" connect {device_id}')
                        time.sleep(0.5)
                        
                        devices.append((device_id, device_info))
                        print(f"已添加设备: {device_info} ({device_id})")
                
                return devices
            except json.JSONDecodeError:
                print("无法将MuMuManager输出解析为JSON，尝试文本解析方式")
            
            # 如果JSON解析失败，回退到原来的文本解析方式
            lines = info_text.strip().split('\n')
            
            current_name = None
            current_index = None
            current_port = None
            
            for line in lines:
                line = line.strip()
                
                # 查找模拟器名称和索引
                if "name:" in line:
                    parts = line.split(':', 1)
                    if len(parts) > 1:
                        current_name = parts[1].strip()
                elif "index:" in line:
                    parts = line.split(':', 1)
                    if len(parts) > 1:
                        try:
                            current_index = int(parts[1].strip())
                        except ValueError:
                            current_index = None
                elif "adb_port:" in line:
                    parts = line.split(':', 1)
                    if len(parts) > 1:
                        try:
                            current_port = int(parts[1].strip())
                        except ValueError:
                            current_port = None
                
                # 当我们有足够的信息时，添加设备
                if current_name and current_port:
                    device_id = f"127.0.0.1:{current_port}"
                    device_info = f"MuMu模拟器 {current_name}"
                    if current_index is not None:
                        device_info += f" (#{current_index})"
                    
                    # 尝试连接设备
                    print(f"尝试连接MuMu设备: {device_id}")
                    os.system(f'"{self.adb_path}" connect {device_id}')
                    time.sleep(0.5)
                    
                    devices.append((device_id, device_info))
                    
                    # 重置当前设备信息
                    current_name = None
                    current_index = None
                    current_port = None
        
        except Exception as e:
            print(f"解析MuMu信息出错: {str(e)}")
            traceback.print_exc()
        
        return devices

def main():
    """主函数"""
    try:
        # 设置全局异常处理
        def report_exception(exc_type, exc_value, exc_traceback):
            """报告异常"""
            logger.error("未捕获的异常", exc_info=(exc_type, exc_value, exc_traceback))
            messagebox.showerror("错误", f"发生未捕获的异常: {exc_value}")
        
        # 保存原始的异常处理器
        original_excepthook = sys.excepthook
        sys.excepthook = report_exception
        
        # 启动主窗口
        root = tk.Tk()
        app = AutoClickUI(root)
        
        # 正常退出时恢复异常处理器
        try:
            root.mainloop()
        finally:
            sys.excepthook = original_excepthook
            
    except Exception as e:
        # 捕获初始化过程中的错误
        logger.error(f"主函数出错: {e}", exc_info=True)
        traceback.print_exc()
        print(f"程序启动时出错: {e}")
        input("按回车键退出...")

if __name__ == "__main__":
    # 确保主线程是tkinter的主线程
    main() 