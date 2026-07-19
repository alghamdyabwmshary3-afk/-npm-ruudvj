import os
import sys
import subprocess
import signal
import time
import threading
import json
import shutil
import psutil
from datetime import datetime
from flask import Flask, render_template_string, request, redirect, jsonify, send_from_directory, session
from werkzeug.utils import secure_filename
from functools import wraps

app = Flask(__name__)
app.secret_key = os.urandom(24)

# الإعدادات المتقدمة
UPLOAD_FOLDER = 'my_bots'
LOGS_FOLDER = 'bot_logs'
BACKUP_FOLDER = 'backups'
CONFIG_FILE = 'config.json'
ALLOWED_EXTENSIONS = {'py', 'txt', 'json', 'sh', 'bat'}

# إنشاء المجلدات
for folder in [UPLOAD_FOLDER, LOGS_FOLDER, BACKUP_FOLDER]:
    os.makedirs(folder, exist_ok=True)

# قواميس التتبع المتقدمة
active_processes = {}
auto_restart_registry = {}
process_metadata = {}
system_stats = {
    'cpu_usage': 0,
    'memory_usage': 0,
    'disk_usage': 0,
    'uptime': time.time()
}

# قفل للتعامل مع العمليات بشكل آمن
process_lock = threading.Lock()

# دوال مساعدة متقدمة
def allowed_file(filename):
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS

def get_process_info(pid):
    """الحصول على معلومات مفصلة عن العملية"""
    try:
        process = psutil.Process(pid)
        return {
            'cpu_percent': process.cpu_percent(interval=0.1),
            'memory_percent': process.memory_percent(),
            'memory_rss': process.memory_info().rss / (1024 * 1024),  # MB
            'status': process.status(),
            'create_time': datetime.fromtimestamp(process.create_time()).strftime('%Y-%m-%d %H:%M:%S')
        }
    except:
        return None

def cleanup_zombie_processes():
    """تنظيف العمليات الميتة بشكل دوري"""
    with process_lock:
        for filename in list(active_processes.keys()):
            process = active_processes[filename]
            if process.poll() is not None:
                del active_processes[filename]
                if filename in auto_restart_registry:
                    auto_restart_registry[filename] = False

# الحارس الذكي المتطور
def daemon_auto_restart_worker():
    """الحارس الذكي لإعادة التشغيل مع مراقبة الأداء"""
    while True:
        try:
            time.sleep(3)
            
            # تنظيف العمليات الميتة
            cleanup_zombie_processes()
            
            # تحديث إحصائيات النظام
            system_stats['cpu_usage'] = psutil.cpu_percent(interval=0.5)
            system_stats['memory_usage'] = psutil.virtual_memory().percent
            system_stats['disk_usage'] = psutil.disk_usage('/').percent
            
            # فحص وإعادة تشغيل البوتات المسجلة
            for filename, need_restart in list(auto_restart_registry.items()):
                if need_restart:
                    file_path = os.path.join(UPLOAD_FOLDER, filename)
                    if not os.path.exists(file_path):
                        continue
                        
                    # التحقق من حالة العملية
                    process = active_processes.get(filename)
                    if process is None or process.poll() is not None:
                        # تسجيل الحدث
                        log_path = os.path.join(LOGS_FOLDER, f"{filename}.log")
                        with open(log_path, "a", encoding="utf-8") as log_file:
                            log_file.write(f"\n[{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}] [DAEMON] إعادة تشغيل تلقائي للبوت {filename}\n")
                        
                        # إعادة التشغيل
                        try:
                            with open(log_path, "a", encoding="utf-8") as log_file:
                                process = subprocess.Popen(
                                    [sys.executable, file_path],
                                    stdout=log_file,
                                    stderr=log_file,
                                    text=True,
                                    preexec_fn=None if os.name == 'nt' else os.setsid
                                )
                                with process_lock:
                                    active_processes[filename] = process
                                    process_metadata[filename] = {
                                        'start_time': datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
                                        'restart_count': process_metadata.get(filename, {}).get('restart_count', 0) + 1,
                                        'pid': process.pid
                                    }
                        except Exception as e:
                            print(f"خطأ في إعادة تشغيل {filename}: {e}")
        
        except Exception as e:
            print(f"خطأ في الحارس الذكي: {e}")
            time.sleep(10)

# تشغيل الحارس
threading.Thread(target=daemon_auto_restart_worker, daemon=True).start()

# واجهة HTML متطورة
HTML_TEMPLATE = '''
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>🚀 المنصة العملاقة للتحكم الشامل بالسيرفر والملفات</title>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    <style>
        :root {
            --bg: #030712;
            --card-bg: rgba(17, 24, 39, 0.9);
            --accent: #00f0ff;
            --purple: #a855f7;
            --pink: #ec4899;
            --text: #f8fafc;
            --text-sub: #94a3b8;
            --green: #10b981;
            --red: #ef4444;
            --yellow: #f59e0b;
            --border: rgba(255, 255, 255, 0.06);
            --shadow: 0 20px 40px rgba(0,0,0,0.5);
        }
        
        * { margin: 0; padding: 0; box-sizing: border-box; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; }
        
        body { 
            background: radial-gradient(circle at center, #0f172a, var(--bg)); 
            color: var(--text); padding: 20px; min-height: 100vh; 
        }
        
        .container { max-width: 1400px; margin: 0 auto; }
        
        /* Header */
        header { 
            text-align: center; margin-bottom: 30px; 
            background: linear-gradient(135deg, rgba(30, 41, 59, 0.6), rgba(3, 7, 18, 0.9)); 
            padding: 40px; border-radius: 24px; border: 1px solid var(--border);
            backdrop-filter: blur(20px); box-shadow: var(--shadow);
            position: relative;
            overflow: hidden;
        }
        
        header::before {
            content: '';
            position: absolute;
            top: -50%;
            left: -50%;
            width: 200%;
            height: 200%;
            background: radial-gradient(circle, rgba(0,240,255,0.03) 0%, transparent 70%);
            animation: rotate 20s linear infinite;
        }
        
        @keyframes rotate {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
        
        header h1 { 
            color: var(--accent); font-size: 2.5rem; margin-bottom: 10px; 
            text-shadow: 0 0 30px rgba(0,240,255,0.3);
            position: relative;
            z-index: 1;
        }
        
        header p { 
            color: var(--text-sub); font-size: 1.1rem; 
            position: relative;
            z-index: 1;
        }
        
        /* Stats Grid */
        .stats-grid { 
            display: grid; 
            grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); 
            gap: 15px; 
            margin-bottom: 25px; 
        }
        
        .stat-card { 
            background: var(--card-bg); 
            border: 1px solid var(--border); 
            border-radius: 16px; 
            padding: 20px; 
            display: flex; 
            align-items: center; 
            justify-content: space-between;
            transition: all 0.3s ease;
            backdrop-filter: blur(10px);
        }
        
        .stat-card:hover {
            transform: translateY(-3px);
            border-color: rgba(0,240,255,0.2);
            box-shadow: 0 10px 25px rgba(0,0,0,0.3);
        }
        
        .stat-info h3 { 
            font-size: 1.8rem; 
            color: var(--accent); 
            font-weight: bold;
        }
        
        .stat-info p { 
            color: var(--text-sub); 
            font-size: 0.8rem; 
            margin-top: 5px;
        }
        
        .stat-icon { 
            font-size: 2.5rem; 
            opacity: 0.2;
            transition: all 0.3s ease;
        }
        
        .stat-card:hover .stat-icon {
            opacity: 0.4;
            transform: scale(1.1);
        }
        
        /* Panels */
        .panel { 
            background: var(--card-bg); 
            border: 1px solid var(--border); 
            border-radius: 20px; 
            padding: 30px; 
            margin-bottom: 25px; 
            box-shadow: var(--shadow);
            backdrop-filter: blur(20px);
        }
        
        .panel-title { 
            font-size: 1.3rem; 
            font-weight: bold; 
            color: var(--accent); 
            margin-bottom: 20px; 
            display: flex; 
            align-items: center; 
            gap: 12px; 
            border-bottom: 1px solid var(--border); 
            padding-bottom: 12px;
        }
        
        /* Upload Area */
        .upload-area { 
            border: 2px dashed #475569; 
            padding: 50px; 
            text-align: center; 
            border-radius: 16px; 
            cursor: pointer; 
            transition: all 0.3s ease; 
            display: block;
            position: relative;
        }
        
        .upload-area:hover { 
            border-color: var(--accent); 
            background: rgba(0,240,255,0.03);
            box-shadow: 0 0 40px rgba(0,240,255,0.1);
            transform: scale(1.01);
        }
        
        .upload-area i { 
            font-size: 4rem; 
            color: var(--accent); 
            margin-bottom: 15px;
            transition: all 0.3s ease;
        }
        
        .upload-area:hover i {
            transform: scale(1.1) rotate(-5deg);
        }
        
        /* Bot Cards */
        .bot-card { 
            display: flex; 
            justify-content: space-between; 
            align-items: center; 
            padding: 20px 25px; 
            background: rgba(255,255,255,0.02); 
            border: 1px solid var(--border); 
            border-radius: 14px; 
            margin-bottom: 12px; 
            flex-wrap: wrap; 
            gap: 15px; 
            transition: all 0.3s ease;
        }
        
        .bot-card:hover { 
            background: rgba(255,255,255,0.04); 
            border-color: rgba(0,240,255,0.15);
            transform: translateX(5px);
        }
        
        .bot-details { 
            display: flex; 
            align-items: center; 
            gap: 15px; 
        }
        
        .bot-details i { 
            font-size: 2.5rem; 
            background: linear-gradient(135deg, #38bdf8, #a855f7);
            -webkit-background-clip: text;
            -webkit-text-fill-color: transparent;
        }
        
        .bot-name { 
            font-size: 1.1rem; 
            font-weight: 600; 
            color: #fff; 
        }
        
        .bot-meta {
            font-size: 0.75rem;
            color: var(--text-sub);
            margin-top: 3px;
        }
        
        .status-badge { 
            display: inline-flex; 
            align-items: center; 
            gap: 8px; 
            padding: 6px 14px; 
            border-radius: 20px; 
            font-size: 0.8rem; 
            font-weight: 600; 
        }
        
        .badge-active { 
            background: rgba(16, 185, 129, 0.15); 
            color: var(--green); 
            border: 1px solid rgba(16, 185, 129, 0.3); 
        }
        
        .badge-inactive { 
            background: rgba(239, 68, 68, 0.1); 
            color: var(--red); 
            border: 1px solid rgba(239, 68, 68, 0.2); 
        }
        
        .badge-restarting {
            background: rgba(245, 158, 11, 0.15);
            color: var(--yellow);
            border: 1px solid rgba(245, 158, 11, 0.3);
        }
        
        .btn { 
            border: none; 
            padding: 10px 18px; 
            border-radius: 10px; 
            font-size: 0.85rem; 
            font-weight: 600; 
            cursor: pointer; 
            display: inline-flex; 
            align-items: center; 
            gap: 8px; 
            color: #fff; 
            transition: all 0.3s ease; 
            text-decoration: none;
            position: relative;
            overflow: hidden;
        }
        
        .btn::after {
            content: '';
            position: absolute;
            top: -50%;
            left: -50%;
            width: 200%;
            height: 200%;
            background: radial-gradient(circle, rgba(255,255,255,0.1) 0%, transparent 70%);
            opacity: 0;
            transition: all 0.3s ease;
        }
        
        .btn:hover::after {
            opacity: 1;
        }
        
        .btn:hover { 
            transform: translateY(-2px); 
            box-shadow: 0 8px 20px rgba(0,0,0,0.3);
        }
        
        .btn:active {
            transform: scale(0.95);
        }
        
        .btn-start { 
            background: linear-gradient(135deg, #059669, #10b981); 
        }
        
        .btn-stop { 
            background: linear-gradient(135deg, #dc2626, #ef4444); 
        }
        
        .btn-download { 
            background: linear-gradient(135deg, #334155, #475569); 
        }
        
        .btn-del { 
            background: rgba(239, 68, 68, 0.1); 
            color: #ef4444; 
            border: 1px solid rgba(239,68,68,0.2); 
        }
        
        .btn-del:hover { 
            background: #ef4444; 
            color: #fff; 
        }
        
        .btn-backup {
            background: linear-gradient(135deg, #7c3aed, #a855f7);
        }
        
        /* Terminal */
        .terminal-box { 
            background: #020617; 
            border: 1px solid rgba(255,255,255,0.05); 
            border-radius: 14px; 
            padding: 20px; 
            font-family: 'Courier New', Courier, monospace; 
            font-size: 0.9rem; 
            color: #34d399; 
            height: 400px; 
            overflow-y: auto; 
            white-space: pre-wrap;
            box-shadow: inset 0 0 30px rgba(0,0,0,0.8);
            line-height: 1.6;
        }
        
        .terminal-box::-webkit-scrollbar {
            width: 8px;
        }
        
        .terminal-box::-webkit-scrollbar-track {
            background: #020617;
        }
        
        .terminal-box::-webkit-scrollbar-thumb {
            background: #1e293b;
            border-radius: 4px;
        }
        
        .terminal-box::-webkit-scrollbar-thumb:hover {
            background: #334155;
        }
        
        .select-log {
            background: #1e293b; 
            color: #fff; 
            border: 1px solid var(--border);
            padding: 12px 18px; 
            border-radius: 10px; 
            font-size: 0.9rem; 
            outline: none; 
            margin-bottom: 15px; 
            width: 100%; 
            max-width: 350px;
            transition: all 0.3s ease;
        }
        
        .select-log:focus {
            border-color: var(--accent);
            box-shadow: 0 0 20px rgba(0,240,255,0.1);
        }
        
        /* Toast notification */
        .toast {
            position: fixed;
            bottom: 30px;
            right: 30px;
            background: var(--card-bg);
            border: 1px solid var(--border);
            border-radius: 12px;
            padding: 15px 25px;
            backdrop-filter: blur(20px);
            box-shadow: var(--shadow);
            transform: translateY(100px);
            opacity: 0;
            transition: all 0.5s ease;
            z-index: 1000;
            max-width: 400px;
        }
        
        .toast.show {
            transform: translateY(0);
            opacity: 1;
        }
        
        .toast i {
            margin-right: 10px;
        }
        
        /* Responsive */
        @media (max-width: 768px) {
            header h1 { font-size: 1.8rem; }
            .bot-card { flex-direction: column; align-items: stretch; }
            .actions-row { justify-content: center; }
            .stats-grid { grid-template-columns: repeat(auto-fit, minmax(150px, 1fr)); }
            .panel { padding: 20px; }
        }
        
        /* Animations */
        @keyframes pulse {
            0%, 100% { opacity: 1; }
            50% { opacity: 0.5; }
        }
        
        .pulsing {
            animation: pulse 2s ease-in-out infinite;
        }
        
        .glow-text {
            text-shadow: 0 0 20px rgba(0,240,255,0.3);
        }
        
        /* Floating particles */
        .particles {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            pointer-events: none;
            z-index: -1;
            overflow: hidden;
        }
        
        .particle {
            position: absolute;
            width: 3px;
            height: 3px;
            background: var(--accent);
            border-radius: 50%;
            opacity: 0.1;
            animation: float linear infinite;
        }
        
        @keyframes float {
            0% {
                transform: translateY(100vh) rotate(0deg);
                opacity: 0;
            }
            10% {
                opacity: 0.1;
            }
            90% {
                opacity: 0.1;
            }
            100% {
                transform: translateY(-100vh) rotate(720deg);
                opacity: 0;
            }
        }
    </style>
</head>
<body>
    <!-- Floating particles -->
    <div class="particles" id="particles"></div>
    
    <div class="container">
        <header>
            <h1><i class="fa-solid fa-shield-halved"></i> المنصة السحابية العملاقة</h1>
            <p>المحرك الأكثر استقراراً وعزلاً لتشغيل أضخم ملفات الذكاء الاصطناعي وبوتات تليجرام دون قيود الموارد</p>
            <div style="margin-top: 15px; font-size: 0.9rem; color: var(--text-sub);">
                <i class="fa-solid fa-microchip"></i> CPU: <span id="cpu-usage">0</span>% &nbsp;|&nbsp;
                <i class="fa-solid fa-memory"></i> RAM: <span id="memory-usage">0</span>% &nbsp;|&nbsp;
                <i class="fa-solid fa-hard-drive"></i> DISK: <span id="disk-usage">0</span>%
            </div>
        </header>

        <!-- Stats Grid -->
        <div class="stats-grid">
            <div class="stat-card">
                <div class="stat-info">
                    <h3 id="total-bots">0</h3>
                    <p><i class="fa-regular fa-file"></i> إجمالي الملفات</p>
                </div>
                <div class="stat-icon"><i class="fa-solid fa-folder-open"></i></div>
            </div>
            <div class="stat-card">
                <div class="stat-info">
                    <h3 id="active-bots" style="color: var(--green);">0</h3>
                    <p><i class="fa-regular fa-circle-play"></i> قيد التشغيل</p>
                </div>
                <div class="stat-icon"><i class="fa-solid fa-play"></i></div>
            </div>
            <div class="stat-card">
                <div class="stat-info">
                    <h3 id="restart-count" style="color: var(--yellow);">0</h3>
                    <p><i class="fa-solid fa-rotate"></i> عمليات إعادة التشغيل</p>
                </div>
                <div class="stat-icon"><i class="fa-solid fa-arrow-rotate-right"></i></div>
            </div>
            <div class="stat-card">
                <div class="stat-info">
                    <h3 style="color: var(--purple);"><i class="fa-solid fa-shield"></i></h3>
                    <p>الحارس الذكي نشط</p>
                </div>
                <div class="stat-icon"><i class="fa-solid fa-user-shield"></i></div>
            </div>
        </div>

        <!-- Upload Panel -->
        <div class="panel">
            <div class="panel-title"><i class="fa-solid fa-cloud-arrow-up"></i> رفع الملفات الذكي</div>
            <form action="/upload" method="post" enctype="multipart/form-data">
                <label class="upload-area" for="fileInput">
                    <i class="fa-solid fa-square-terminal"></i>
                    <p style="font-size: 1.1rem; margin: 10px 0;">انقر هنا أو اسحب الملف للرفع</p>
                    <p style="color: var(--text-sub); font-size: 0.85rem;">الملفات المدعومة: .py, .txt, .json, .sh, .bat</p>
                </label>
                <input type="file" id="fileInput" name="file" accept=".py,.txt,.json,.sh,.bat" onchange="this.form.submit()" style="display:none;">
            </form>
        </div>

        <!-- Bot Management -->
        <div class="panel">
            <div class="panel-title"><i class="fa-solid fa-server"></i> إدارة العمليات</div>
            <div>
                {% if not bots %}
                <p style="text-align: center; color: var(--text-sub); padding: 20px 0;">
                    <i class="fa-regular fa-inbox" style="font-size: 2rem; display: block; margin-bottom: 10px;"></i>
                    لا توجد ملفات مخزنة حالياً
                </p>
                {% endif %}
                {% for bot in bots %}
                <div class="bot-card" data-bot="{{ bot.name }}">
                    <div class="bot-details">
                        <i class="fa-solid fa-box-archive"></i>
                        <div>
                            <div class="bot-name">{{ bot.name }}</div>
                            <div class="bot-meta">
                                {% if bot.running %}
                                <span style="color: var(--green);">●</span> يعمل
                                {% else %}
                                <span style="color: var(--red);">●</span> متوقف
                                {% endif %}
                                {% if bot.metadata %}
                                &nbsp;|&nbsp; <i class="fa-regular fa-clock"></i> {{ bot.metadata.start_time or 'غير معروف' }}
                                {% endif %}
                            </div>
                        </div>
                    </div>
                    <div>
                        {% if bot.running %}
                        <span class="status-badge badge-active"><i class="fa-solid fa-bolt"></i> نشط</span>
                        {% else %}
                        <span class="status-badge badge-inactive"><i class="fa-solid fa-power-off"></i> متوقف</span>
                        {% endif %}
                    </div>
                    <div class="actions-row">
                        {% if not bot.running %}
                        <a href="/start/{{ bot.name }}" class="btn btn-start" onclick="showToast('جاري تشغيل {{ bot.name }}...')">
                            <i class="fa-solid fa-play"></i> تشغيل
                        </a>
                        {% else %}
                        <a href="/stop/{{ bot.name }}" class="btn btn-stop" onclick="showToast('جاري إيقاف {{ bot.name }}...')">
                            <i class="fa-solid fa-circle-stop"></i> إيقاف
                        </a>
                        {% endif %}
                        <a href="/download/{{ bot.name }}" class="btn btn-download"><i class="fa-solid fa-download"></i></a>
                        <a href="/backup/{{ bot.name }}" class="btn btn-backup"><i class="fa-solid fa-floppy-disk"></i></a>
                        <a href="/delete/{{ bot.name }}" class="btn btn-del" onclick="return confirm('هل أنت متأكد من حذف {{ bot.name }}؟')">
                            <i class="fa-solid fa-trash-can"></i>
                        </a>
                    </div>
                </div>
                {% endfor %}
            </div>
        </div>

        <!-- Console -->
        <div class="panel">
            <div class="panel-title"><i class="fa-solid fa-rectangle-terminal"></i> وحدة التحكم والمخرجات</div>
            <select id="botLogSelect" class="select-log" onchange="fetchLogs()">
                <option value="">-- اختر السكربت لعرض سجلاته --</option>
                {% for bot in bots %}
                <option value="{{ bot.name }}">{{ bot.name }}</option>
                {% endfor %}
            </select>
            <div class="terminal-box" id="terminalConsole">
[SYSTEM] المنصة جاهزة للعمل...
[SYSTEM] اختر سكربت لعرض مخرجاته في الوقت الفعلي
            </div>
        </div>
    </div>

    <!-- Toast Notification -->
    <div class="toast" id="toast">
        <i class="fa-solid fa-circle-check" style="color: var(--green);"></i>
        <span id="toast-message">تم تنفيذ العملية بنجاح</span>
    </div>

    <script>
        // تهيئة الجسيمات العائمة
        function initParticles() {
            const container = document.getElementById('particles');
            for (let i = 0; i < 30; i++) {
                const particle = document.createElement('div');
                particle.className = 'particle';
                particle.style.left = Math.random() * 100 + '%';
                particle.style.animationDuration = (20 + Math.random() * 30) + 's';
                particle.style.animationDelay = (Math.random() * 20) + 's';
                particle.style.width = (2 + Math.random() * 4) + 'px';
                particle.style.height = particle.style.width;
                container.appendChild(particle);
            }
        }
        initParticles();

        // تحديث الإحصائيات
        function updateStats() {
            const cards = document.querySelectorAll('.bot-card');
            const activeCards = document.querySelectorAll('.badge-active');
            document.getElementById('total-bots').innerText = cards.length;
            document.getElementById('active-bots').innerText = activeCards.length;
            
            // حساب عدد عمليات إعادة التشغيل
            let restartCount = 0;
            document.querySelectorAll('.bot-card').forEach(card => {
                const meta = card.querySelector('.bot-meta');
                if (meta && meta.textContent.includes('إعادة التشغيل')) {
                    restartCount++;
                }
            });
            document.getElementById('restart-count').innerText = restartCount;
        }
        
        // جلب السجلات
        function fetchLogs() {
            const selectedBot = document.getElementById('botLogSelect').value;
            const consoleDiv = document.getElementById('terminalConsole');
            if (!selectedBot) {
                consoleDiv.innerText = '[SYSTEM] اختر سكربت لعرض مخرجاته...';
                return;
            }
            
            fetch('/logs/' + selectedBot)
                .then(response => response.json())
                .then(data => {
                    consoleDiv.innerText = data.logs || '[SYSTEM] لا توجد سجلات لهذا السكربت بعد';
                    consoleDiv.scrollTop = consoleDiv.scrollHeight;
                })
                .catch(() => {
                    consoleDiv.innerText = '[ERROR] تعذر جلب السجلات';
                });
        }
        
        // تحديث إحصائيات النظام
        function updateSystemStats() {
            fetch('/system_stats')
                .then(response => response.json())
                .then(data => {
                    document.getElementById('cpu-usage').innerText = data.cpu;
                    document.getElementById('memory-usage').innerText = data.memory;
                    document.getElementById('disk-usage').innerText = data.disk;
                });
        }
        
        // عرض إشعار
        function showToast(message, isError = false) {
            const toast = document.getElementById('toast');
            const toastMessage = document.getElementById('toast-message');
            toastMessage.innerText = message;
            toast.style.borderColor = isError ? 'var(--red)' : 'var(--green)';
            toast.querySelector('i').style.color = isError ? 'var(--red)' : 'var(--green)';
            toast.querySelector('i').className = isError ? 'fa-solid fa-circle-exclamation' : 'fa-solid fa-circle-check';
            toast.classList.add('show');
            setTimeout(() => toast.classList.remove('show'), 3000);
        }
        
        // تحديث تلقائي
        setInterval(() => {
            updateStats();
            updateSystemStats();
            if (document.getElementById('botLogSelect').value) {
                fetchLogs();
            }
        }, 2000);
        
        // تحديث أولي
        updateStats();
        updateSystemStats();
    </script>
</body>
</html>
'''

# Routes
@app.route('/')
def index():
    files = os.listdir(UPLOAD_FOLDER)
    bots_status = []
    for f in files:
        if allowed_file(f):
            is_running = False
            metadata = process_metadata.get(f, {})
            
            if f in active_processes:
                process = active_processes[f]
                if process.poll() is None:
                    is_running = True
                else:
                    with process_lock:
                        if f in active_processes:
                            del active_processes[f]
            
            bots_status.append({
                'name': f,
                'running': is_running,
                'metadata': metadata
            })
    
    return render_template_string(HTML_TEMPLATE, bots=bots_status)

@app.route('/upload', methods=['POST'])
def upload_file():
    if 'file' not in request.files:
        return redirect('/')
    
    file = request.files['file']
    if file.filename == '':
        return redirect('/')
    
    if not allowed_file(file.filename):
        return "نوع الملف غير مدعوم", 400
    
    filename = secure_filename(file.filename)
    file_path = os.path.join(UPLOAD_FOLDER, filename)
    
    # عمل نسخة احتياطية إذا كان الملف موجود
    if os.path.exists(file_path):
        backup_path = os.path.join(BACKUP_FOLDER, f"{filename}.{int(time.time())}.bak")
        shutil.copy2(file_path, backup_path)
    
    file.save(file_path)
    return redirect('/')

@app.route('/start/<filename>')
def start_bot(filename):
    if not allowed_file(filename):
        return "نوع الملف غير مدعوم", 400
    
    file_path = os.path.join(UPLOAD_FOLDER, filename)
    if not os.path.exists(file_path):
        return "الملف غير موجود", 404
    
    with process_lock:
        # إيقاف العملية القديمة إذا كانت موجودة
        if filename in active_processes:
            process = active_processes[filename]
            if process.poll() is None:
                try:
                    if os.name == 'nt':
                        process.terminate()
                    else:
                        os.killpg(os.getpgid(process.pid), signal.SIGTERM)
                except:
                    process.kill()
            del active_processes[filename]
        
        # تشغيل العملية الجديدة
        log_path = os.path.join(LOGS_FOLDER, f"{filename}.log")
        with open(log_path, "w", encoding="utf-8") as log_file:
            log_file.write(f"[{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}] بدء تشغيل {filename}\n")
        
        try:
            with open(log_path, "a", encoding="utf-8") as log_file:
                process = subprocess.Popen(
                    [sys.executable, file_path],
                    stdout=log_file,
                    stderr=log_file,
                    text=True,
                    preexec_fn=None if os.name == 'nt' else os.setsid
                )
                active_processes[filename] = process
                auto_restart_registry[filename] = True
                process_metadata[filename] = {
                    'start_time': datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
                    'restart_count': process_metadata.get(filename, {}).get('restart_count', 0),
                    'pid': process.pid
                }
        except Exception as e:
            return f"خطأ في تشغيل البوت: {e}", 500
    
    return redirect('/')

@app.route('/stop/<filename>')
def stop_bot(filename):
    with process_lock:
        if filename in auto_restart_registry:
            auto_restart_registry[filename] = False
        
        if filename in active_processes:
            process = active_processes[filename]
            if process.poll() is None:
                try:
                    if os.name == 'nt':
                        process.terminate()
                    else:
                        os.killpg(os.getpgid(process.pid), signal.SIGTERM)
                    time.sleep(0.5)
                    if process.poll() is None:
                        process.kill()
                except:
                    process.kill()
            del active_processes[filename]
            
            # تسجيل التوقف
            log_path = os.path.join(LOGS_FOLDER, f"{filename}.log")
            with open(log_path, "a", encoding="utf-8") as log_file:
                log_file.write(f"[{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}] تم إيقاف {filename}\n")
    
    return redirect('/')

@app.route('/download/<filename>')
def download_bot(filename):
    if not allowed_file(filename):
        return "نوع الملف غير مدعوم", 400
    return send_from_directory(UPLOAD_FOLDER, filename, as_attachment=True)

@app.route('/backup/<filename>')
def backup_bot(filename):
    if not allowed_file(filename):
        return "نوع الملف غير مدعوم", 400
    
    source_path = os.path.join(UPLOAD_FOLDER, filename)
    if not os.path.exists(source_path):
        return "الملف غير موجود", 404
    
    backup_name = f"{filename}.{int(time.time())}.bak"
    backup_path = os.path.join(BACKUP_FOLDER, backup_name)
    shutil.copy2(source_path, backup_path)
    
    return redirect('/')

@app.route('/delete/<filename>')
def delete_bot(filename):
    with process_lock:
        if filename in auto_restart_registry:
            del auto_restart_registry[filename]
        
        if filename in active_processes:
            try:
                process = active_processes[filename]
                if process.poll() is None:
                    if os.name == 'nt':
                        process.terminate()
                    else:
                        os.killpg(os.getpgid(process.pid), signal.SIGTERM)
                    time.sleep(0.5)
                    if process.poll() is None:
                        process.kill()
            except:
                pass
            del active_processes[filename]
        
        if filename in process_metadata:
            del process_metadata[filename]
    
    # حذف الملفات
    file_path = os.path.join(UPLOAD_FOLDER, filename)
    log_path = os.path.join(LOGS_FOLDER, f"{filename}.log")
    
    if os.path.exists(file_path):
        os.remove(file_path)
    if os.path.exists(log_path):
        try:
            os.remove(log_path)
        except:
            pass
    
    return redirect('/')

@app.route('/logs/<filename>')
def get_logs(filename):
    if not allowed_file(filename):
        return jsonify({'logs': 'نوع الملف غير مدعوم'})
    
    log_path = os.path.join(LOGS_FOLDER, f"{filename}.log")
    if os.path.exists(log_path):
        try:
            with open(log_path, "r", encoding="utf-8", errors="ignore") as f:
                content = f.read()
                # جلب آخر 10000 حرف
                return jsonify({'logs': content[-10000:]})
        except:
            return jsonify({'logs': 'خطأ في قراءة السجل'})
    
    return jsonify({'logs': 'لا توجد سجلات بعد'})

@app.route('/system_stats')
def get_system_stats():
    return jsonify({
        'cpu': system_stats['cpu_usage'],
        'memory': system_stats['memory_usage'],
        'disk': system_stats['disk_usage']
    })

@app.route('/process_info/<filename>')
def get_process_info(filename):
    if filename not in active_processes:
        return jsonify({'error': 'العملية غير موجودة'})
    
    process = active_processes[filename]
    if process.poll() is not None:
        return jsonify({'error': 'العملية متوقفة'})
    
    info = get_process_info(process.pid)
    if info:
        return jsonify(info)
    return jsonify({'error': 'لا يمكن الحصول على معلومات'})

# Error handlers
@app.errorhandler(404)
def not_found(error):
    return render_template_string('''
        <div style="text-align: center; padding: 50px;">
            <h1 style="color: var(--red);">404</h1>
            <p>الصفحة غير موجودة</p>
            <a href="/" style="color: var(--accent);">العودة للرئيسية</a>
        </div>
    '''), 404

@app.errorhandler(500)
def internal_error(error):
    return render_template_string('''
        <div style="text-align: center; padding: 50px;">
            <h1 style="color: var(--red);">500</h1>
            <p>خطأ داخلي في السيرفر</p>
            <a href="/" style="color: var(--accent);">العودة للرئيسية</a>
        </div>
    '''), 500

if __name__ == '__main__':
    print("""
    ╔═══════════════════════════════════════════╗
    ║   🚀 المنصة العملاقة للتشغيل والتحكم    ║
    ║   -----------------------------------    ║
    ║   • تشغيل متعدد العمليات                ║
    ║   • مراقبة لحظية للأداء                 ║
    ║   • إعادة تشغيل تلقائي ذكي              ║
    ║   • نظام احتياطي متكامل                 ║
    ║   • واجهة سحابية تفاعلية                ║
    ╚═══════════════════════════════════════════╝
    """)
    app.run(host='0.0.0.0', port=8080, debug=False, threaded=True)
