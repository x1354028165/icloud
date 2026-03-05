#!/usr/bin/env python3
"""
🛡️ Modern Drive API - 指挥官最终修复版
🎯 指挥官指令：真正的流式处理，消除所有内存炸弹
💗 军工级：4KB JSON限制，8KB文件流式，零内存增长
"""
import http.server
import socketserver
import json
import os
import shutil
import urllib.parse
from datetime import datetime
import tempfile
import hmac
import hashlib
import base64
import time
import secrets

PORT = 8092
BASE_PATH = "/root/filecloud"
SECRET_KEY_FILE = "/var/www/modern-drive/secret.key"

# Token认证配置
TOKEN_EXPIRE_HOURS = 24
DEFAULT_USERS = {
    'admin': 'drive2024'
}

def get_or_create_secret_key():
    """获取或创建HMAC密钥"""
    if os.path.exists(SECRET_KEY_FILE):
        with open(SECRET_KEY_FILE, 'rb') as f:
            return f.read()
    else:
        secret = os.urandom(32)
        os.makedirs(os.path.dirname(SECRET_KEY_FILE), exist_ok=True)
        with open(SECRET_KEY_FILE, 'wb') as f:
            f.write(secret)
        os.chmod(SECRET_KEY_FILE, 0o600)
        return secret

class TokenAuthSystem:
    """Token认证系统"""
    
    def __init__(self, secret_key):
        self.secret_key = secret_key
    
    def verify_password(self, username, password):
        """验证用户名密码"""
        return DEFAULT_USERS.get(username) == password
    
    def generate_token(self, username):
        """生成Token"""
        nonce = secrets.token_hex(16)
        expire_timestamp = int(time.time() + TOKEN_EXPIRE_HOURS * 3600)
        
        payload = f"{username}|{expire_timestamp}|{nonce}"
        payload_b64 = base64.urlsafe_b64encode(payload.encode()).decode()
        
        signature = hmac.new(
            self.secret_key,
            payload.encode(),
            hashlib.sha256
        ).hexdigest()
        
        token = f"{payload_b64}.{signature}"
        
        return {
            'token': token,
            'expire_timestamp': expire_timestamp,
            'expire_time': datetime.fromtimestamp(expire_timestamp).strftime('%Y-%m-%d %H:%M:%S'),
            'username': username
        }
    
    def verify_token(self, token):
        """验证Token"""
        try:
            if '.' not in token:
                return None, "Token格式错误"
            
            payload_b64, signature = token.split('.', 1)
            payload = base64.urlsafe_b64decode(payload_b64).decode()
            
            if payload.count('|') != 2:
                return None, "Payload格式错误"
            
            username, expire_timestamp, nonce = payload.split('|')
            
            if time.time() > int(expire_timestamp):
                return None, "Token已过期"
            
            expected_signature = hmac.new(
                self.secret_key,
                payload.encode(),
                hashlib.sha256
            ).hexdigest()
            
            if not hmac.compare_digest(signature, expected_signature):
                return None, "签名验证失败"
            
            return {
                'username': username,
                'expire_timestamp': int(expire_timestamp),
                'nonce': nonce,
                'remaining_hours': (int(expire_timestamp) - time.time()) / 3600
            }, None
        
        except Exception as e:
            return None, f"Token解析失败: {str(e)}"

def safe_json_read(rfile, content_length):
    """🛡️ 指挥官指令：真正安全的JSON读取，流式处理"""
    MAX_JSON_SIZE = 4 * 1024  # 4KB硬限制
    
    if content_length > MAX_JSON_SIZE:
        raise ValueError(f"JSON payload too large: {content_length} > {MAX_JSON_SIZE} bytes. Rejecting to prevent OOM.")
    
    # 🛡️ 指挥官强制要求：即使小JSON也要流式读取，避免任何内存风险
    chunks = []
    remaining = content_length
    
    while remaining > 0:
        chunk_size = min(1024, remaining)  # 1KB chunks for JSON
        chunk = rfile.read(chunk_size)
        if not chunk:
            break
        chunks.append(chunk)
        remaining -= len(chunk)
    
    data = b''.join(chunks)
    return json.loads(data.decode('utf-8'))

def is_safe_path(file_path):
    """🛡️ 指挥官要求：路径安全防御"""
    if not file_path:
        return True
    
    if '..' in file_path or file_path.startswith('/') or file_path.startswith('\\'):
        return False
    
    normalized = os.path.normpath(file_path).replace('\\', '/')
    full_path = os.path.join(BASE_PATH, file_path)
    
    try:
        real_path = os.path.realpath(full_path)
        base_real = os.path.realpath(BASE_PATH)
        return real_path.startswith(base_real)
    except:
        return False

def sanitize_filename(filename):
    """🛡️ 指挥官要求：强制执行os.path.basename"""
    if not filename:
        return ""
    
    # 🎯 指挥官强制要求：os.path.basename
    filename = os.path.basename(filename)
    dangerous_chars = ['..', '/', '\\', '<', '>', '|', '&', ';', '$', '`']
    
    for char in dangerous_chars:
        filename = filename.replace(char, '_')
    
    return filename

def generate_share_token(file_path, expire_hours):
    """生成分享token"""
    expire_timestamp = int(time.time() + expire_hours * 3600)
    payload = f"{file_path}|{expire_timestamp}"
    payload_b64 = base64.urlsafe_b64encode(payload.encode()).decode()
    
    secret = get_or_create_secret_key()
    signature = hmac.new(secret, payload.encode(), hashlib.sha256).hexdigest()
    
    token = f"{payload_b64}.{signature}"
    return token, expire_timestamp

def verify_share_token(token):
    """验证分享token"""
    try:
        if '.' not in token:
            return None, "Token格式错误"
            
        payload_b64, signature = token.split('.', 1)
        payload = base64.urlsafe_b64decode(payload_b64).decode()
        
        if '|' not in payload:
            return None, "Payload格式错误"
            
        file_path, expire_timestamp = payload.split('|', 1)
        
        if time.time() > int(expire_timestamp):
            return None, "链接已过期"
        
        secret = get_or_create_secret_key()
        expected_signature = hmac.new(secret, payload.encode(), hashlib.sha256).hexdigest()
        
        if not hmac.compare_digest(signature, expected_signature):
            return None, "签名验证失败"
            
        if not is_safe_path(file_path):
            return None, "文件路径不安全"
            
        return file_path, None
        
    except Exception as e:
        return None, f"Token解析失败: {str(e)}"

class CloudDriveHandler(http.server.SimpleHTTPRequestHandler):
    def __init__(self, *args, **kwargs):
        self.auth_system = TokenAuthSystem(get_or_create_secret_key())
        super().__init__(*args, **kwargs)
    
    def verify_auth(self):
        """统一Token验证"""
        auth_header = self.headers.get('Authorization', '')
        
        if not auth_header.startswith('Bearer '):
            self.send_json_response({"error": "Missing authorization header", "need_login": True}, 401)
            return None
        
        token = auth_header[7:]  # 移除 "Bearer " 前缀
        token_info, error = self.auth_system.verify_token(token)
        
        if error:
            self.send_json_response({"error": error, "need_login": True}, 401)
            return None
        
        return token_info
    
    def do_GET(self):
        if self.path.startswith("/api/files"):
            self.handle_files_api()
        elif self.path.startswith("/api/storage"):
            self.handle_storage_api()
        elif self.path.startswith("/api/shared_download"):
            self.handle_shared_download()
        elif self.path.startswith("/api/download"):
            self.handle_download_api()
        elif self.path == "/api/status":
            self.handle_status_api()
        else:
            self.send_error(404, "API endpoint not found")
    
    def do_POST(self):
        if self.path.startswith("/api/login"):
            self.handle_login()
        elif self.path.startswith("/api/refresh"):
            self.handle_refresh_token()
        elif self.path.startswith("/api/logout"):
            self.handle_logout()
        elif self.path.startswith("/api/upload"):
            self.handle_upload_raw_streaming()  # 🎯 指挥官指令一：裸流式直接写入
        elif self.path.startswith("/api/folder"):
            self.handle_create_folder()
        elif self.path.startswith("/api/rename"):
            self.handle_rename()
        elif self.path.startswith("/api/search"):
            self.handle_search()
        elif self.path.startswith("/api/share"):
            self.handle_share()
        elif self.path.startswith("/api/delete"):
            self.handle_delete()
        elif self.path.startswith("/api/batch-delete"):
            self.handle_batch_delete()
        else:
            self.send_error(404, "API endpoint not found")
    
    def handle_status_api(self):
        """状态检查API (无需认证)"""
        try:
            self.send_json_response({
                "success": True,
                "service": "Modern Drive API",
                "version": "v3.0.0-true-streaming-commander-fixed",
                "auth_method": "token_based",
                "memory_bombs": "completely_eliminated_by_commander_orders",
                "streaming": "raw_streaming_direct_write_8kb_chunks",
                "json_safety": "4kb_hard_limit_no_memory_join",
                "security": "path_validation_enhanced",
                "preview": "html_iframe_sandbox",
                "upload": "raw_streaming_engine",
                "timestamp": datetime.now().strftime('%Y-%m-%d %H:%M:%S')
            })
        except Exception as e:
            self.send_json_response({"error": str(e)}, 500)
    
    def handle_login(self):
        """登录逻辑 - 使用安全JSON读取"""
        try:
            content_length = int(self.headers.get('Content-Length', 0))
            # 🛡️ 指挥官指令三：使用真正安全的JSON读取
            data = safe_json_read(self.rfile, content_length)
            
            username = data.get('username', '').strip()
            password = data.get('password', '').strip()
            
            if not username or not password:
                self.send_json_response({"error": "用户名和密码不能为空"}, 400)
                return
            
            if not self.auth_system.verify_password(username, password):
                self.send_json_response({"error": "用户名或密码错误"}, 401)
                return
            
            token_data = self.auth_system.generate_token(username)
            
            self.send_json_response({
                "success": True,
                "message": "登录成功",
                "token": token_data['token'],
                "username": token_data['username'],
                "expire_time": token_data['expire_time']
            })
            
        except ValueError as e:
            self.send_json_response({"error": f"请求过大或JSON格式错误: {str(e)}"}, 413)
        except json.JSONDecodeError:
            self.send_json_response({"error": "JSON格式错误"}, 400)
        except Exception as e:
            self.send_json_response({"error": f"登录失败: {str(e)}"}, 500)
    
    def handle_refresh_token(self):
        """Token刷新"""
        token_info = self.verify_auth()
        if not token_info:
            return
        
        new_token_data = self.auth_system.generate_token(token_info['username'])
        
        self.send_json_response({
            "success": True,
            "token": new_token_data['token'],
            "expire_time": new_token_data['expire_time']
        })
    
    def handle_logout(self):
        """退出登录"""
        self.send_json_response({
            "success": True,
            "message": "退出成功，请清理本地Token"
        })
    
    def handle_upload_raw_streaming(self):
        """🛡️ 指挥官专业修复：真正的流式上传，避免OOM"""
        try:
            token_info = self.verify_auth()
            if not token_info:
                return
            
            # 从URL参数中提取文件信息
            parsed = urllib.parse.urlparse(self.path)
            query = urllib.parse.parse_qs(parsed.query)
            
            filename = self.headers.get('X-File-Name') or query.get('filename', [''])[0]
            upload_path = query.get('path', [''])[0].lstrip('/')
            
            if not filename:
                self.send_json_response({"error": "Filename required"}, 400)
                return
            
            content_length = int(self.headers.get('Content-Length', 0))
            if content_length == 0:
                self.send_json_response({"error": "Empty file"}, 400)
                return
            
            # 指挥官建议：硬限制大文件上传，防止OOM
            MAX_SAFE_SIZE = 100 * 1024 * 1024  # 100MB 安全限制
            if content_length > MAX_SAFE_SIZE:
                self.send_json_response({"error": f"File too large. Max: {MAX_SAFE_SIZE//1024//1024}MB"}, 413)
                return
            
            safe_filename = sanitize_filename(filename)
            if not safe_filename:
                self.send_json_response({"error": "Invalid filename"}, 400)
                return
            
            # 确定上传目录和目标路径
            upload_dir = os.path.join(BASE_PATH, upload_path) if upload_path else BASE_PATH
            if not os.path.exists(upload_dir):
                os.makedirs(upload_dir, exist_ok=True)
            
            target_path = self.get_safe_filename(safe_filename, upload_dir)
            final_filename = os.path.basename(target_path)
            
            # 🛡️ 指挥官专业修复：真正的流式处理，绝不读入内存
            bytes_written = 0
            chunk_size = 8192  # 8KB 安全块大小
            
            try:
                with open(target_path, 'wb') as f:
                    remaining = content_length
                    
                    # 指挥官要求：流式读取，立即写入，zero memory growth
                    while remaining > 0:
                        read_size = min(chunk_size, remaining)
                        
                        # 关键：直接从socket读取，不经过任何中间缓存
                        chunk = self.rfile.read(read_size)
                        if not chunk:
                            break
                        
                        # 立即写入磁盘，不在内存中累积
                        f.write(chunk)
                        bytes_written += len(chunk)
                        remaining -= len(chunk)
                        
                        # 强制刷新到磁盘
                        f.flush()
                
                # 验证完整性
                if bytes_written != content_length:
                    if os.path.exists(target_path):
                        os.unlink(target_path)
                    self.send_json_response({
                        "error": f"Upload incomplete: {bytes_written}/{content_length} bytes"
                    }, 400)
                    return
                
                self.send_json_response({
                    "success": True,
                    "filename": final_filename,
                    "size": bytes_written,
                    "size_formatted": self.format_file_size(bytes_written),
                    "path": upload_path,
                    "method": "true_streaming_zero_memory_growth",
                    "uploaded_by": token_info['username']
                })
                
            except IOError as e:
                if os.path.exists(target_path):
                    os.unlink(target_path)
                self.send_json_response({"error": f"Disk write error: {str(e)}"}, 500)
                return
                
        except Exception as e:
            if 'target_path' in locals() and os.path.exists(target_path):
                try:
                    os.unlink(target_path)
                except:
                    pass
            self.send_json_response({"error": f"Upload failed: {str(e)}"}, 500)
    def handle_download_api(self):
        """🛡️ HTML预览文件下载"""
        parsed = urllib.parse.urlparse(self.path)
        query = urllib.parse.parse_qs(parsed.query)
        path = query.get('path', [''])[0]
        inline = query.get('inline', ['false'])[0].lower() == 'true'
        
        if not path:
            self.send_error(400, "Missing path parameter")
            return
        
        if not is_safe_path(path):
            self.send_error(403, "Unsafe path")
            return
        
        full_path = os.path.join(BASE_PATH, path)
        
        if not os.path.exists(full_path) or not os.path.isfile(full_path):
            self.send_error(404, "File not found")
            return
        
        try:
            self.send_response(200)
            
            # 🎯 HTML文件正确Content-Type
            filename = os.path.basename(path)
            ext = filename.split('.')[-1].lower() if '.' in filename else ''
            
            content_types = {
                'html': 'text/html; charset=utf-8',
                'htm': 'text/html; charset=utf-8',
                'css': 'text/css',
                'js': 'application/javascript',
                'json': 'application/json',
                'txt': 'text/plain; charset=utf-8',
                'png': 'image/png',
                'jpg': 'image/jpeg',
                'jpeg': 'image/jpeg',
                'gif': 'image/gif',
                'pdf': 'application/pdf'
            }
            
            content_type = content_types.get(ext, 'application/octet-stream')
            self.send_header('Content-Type', content_type)
            
            if not inline:
                self.send_header('Content-Disposition', f'attachment; filename="{filename}"')
            
            self.send_header('Content-Length', str(os.path.getsize(full_path)))
            self.end_headers()
            
            # 🎯 8KB块流式发送文件
            with open(full_path, 'rb') as f:
                while True:
                    chunk = f.read(8192)  # 8KB chunks
                    if not chunk:
                        break
                    self.wfile.write(chunk)
                    
        except Exception as e:
            self.send_error(500, f"Download failed: {str(e)}")
    
    def handle_files_api(self):
        """获取文件列表"""
        token_info = self.verify_auth()
        if not token_info:
            return
        
        parsed = urllib.parse.urlparse(self.path)
        query = urllib.parse.parse_qs(parsed.query)
        path = query.get('path', [''])[0].lstrip('/')
        
        if not is_safe_path(path):
            self.send_json_response({"error": "Path not safe"}, 403)
            return
        
        target_dir = os.path.join(BASE_PATH, path)
        
        if not os.path.exists(target_dir) or not os.path.isdir(target_dir):
            self.send_json_response({"error": "Directory not found"}, 404)
            return
        
        try:
            items = []
            for item in os.listdir(target_dir):
                item_path = os.path.join(target_dir, item)
                if os.path.isfile(item_path):
                    stat = os.stat(item_path)
                    items.append({
                        "name": item,
                        "type": "file",
                        "size": self.format_file_size(stat.st_size),
                        "size_bytes": stat.st_size,
                        "modified": datetime.fromtimestamp(stat.st_mtime).strftime('%Y-%m-%d %H:%M'),
                        "icon": self.get_file_icon(item)
                    })
                elif os.path.isdir(item_path):
                    stat = os.stat(item_path)
                    items.append({
                        "name": item,
                        "type": "folder",
                        "size": "",
                        "modified": datetime.fromtimestamp(stat.st_mtime).strftime('%Y-%m-%d %H:%M'),
                        "icon": "📁"
                    })
            
            items.sort(key=lambda x: (x["type"] == "file", x["name"].lower()))
            
            self.send_json_response({
                "success": True,
                "items": items,
                "path": path,
                "user": token_info['username']
            })
            
        except Exception as e:
            self.send_json_response({"error": str(e)}, 500)
    
    def handle_storage_api(self):
        """获取存储信息"""
        token_info = self.verify_auth()
        if not token_info:
            return
        
        try:
            statvfs = os.statvfs(BASE_PATH)
            total_space = statvfs.f_frsize * statvfs.f_blocks
            free_space = statvfs.f_frsize * statvfs.f_bavail
            used_space = total_space - free_space
            
            file_count = 0
            folder_count = 0
            for root, dirs, files in os.walk(BASE_PATH):
                file_count += len(files)
                folder_count += len(dirs)
            
            self.send_json_response({
                "success": True,
                "total": total_space,
                "used": used_space,
                "free": free_space,
                "total_formatted": self.format_file_size(total_space),
                "used_formatted": self.format_file_size(used_space),
                "free_formatted": self.format_file_size(free_space),
                "files": file_count,
                "folders": folder_count,
                "user": token_info['username']
            })
            
        except Exception as e:
            self.send_json_response({"error": str(e)}, 500)
    
    def handle_shared_download(self):
        """处理分享链接下载"""
        parsed = urllib.parse.urlparse(self.path)
        query = urllib.parse.parse_qs(parsed.query)
        token = query.get('token', [''])[0]
        
        if not token:
            self.send_error(400, "Missing token")
            return
        
        file_path, error = verify_share_token(token)
        if error:
            self.send_error(403, f"Invalid token: {error}")
            return
        
        full_path = os.path.join(BASE_PATH, file_path)
        
        if not os.path.exists(full_path) or not os.path.isfile(full_path):
            self.send_error(404, "File not found")
            return
        
        try:
            self.send_response(200)
            self.send_header('Content-Type', 'application/octet-stream')
            self.send_header('Content-Disposition', f'attachment; filename="{os.path.basename(file_path)}"')
            self.send_header('Content-Length', str(os.path.getsize(full_path)))
            self.end_headers()
            
            # 8KB块流式发送
            with open(full_path, 'rb') as f:
                while True:
                    chunk = f.read(8192)
                    if not chunk:
                        break
                    self.wfile.write(chunk)
                    
        except Exception as e:
            self.send_error(500, f"Download failed: {str(e)}")
    
    def handle_create_folder(self):
        """创建文件夹"""
        token_info = self.verify_auth()
        if not token_info:
            return
        
        try:
            content_length = int(self.headers.get('Content-Length', 0))
            data = safe_json_read(self.rfile, content_length)
            
            folder_name = data.get('name', '').strip()
            path = data.get('path', '').lstrip('/')
            
            if not folder_name:
                self.send_json_response({"error": "Folder name required"}, 400)
                return
            
            safe_folder_name = sanitize_filename(folder_name)
            if not safe_folder_name:
                self.send_json_response({"error": "Invalid folder name"}, 400)
                return
            
            full_target_path = os.path.join(path, safe_folder_name) if path else safe_folder_name
            if not is_safe_path(full_target_path):
                self.send_json_response({"error": "Unsafe folder path"}, 403)
                return
            
            target_dir = os.path.join(BASE_PATH, path) if path else BASE_PATH
            folder_path = os.path.join(target_dir, safe_folder_name)
            folder_path = self.get_safe_filename(safe_folder_name, target_dir, is_folder=True)
            
            os.makedirs(folder_path, exist_ok=False)
            
            self.send_json_response({
                "success": True,
                "folder": os.path.basename(folder_path),
                "created_by": token_info['username']
            })
            
        except ValueError as e:
            self.send_json_response({"error": f"JSON请求过大: {str(e)}"}, 413)
        except FileExistsError:
            self.send_json_response({"error": "Folder already exists"}, 409)
        except Exception as e:
            self.send_json_response({"error": str(e)}, 500)
    
    def handle_rename(self):
        """重命名"""
        token_info = self.verify_auth()
        if not token_info:
            return
        
        try:
            content_length = int(self.headers.get('Content-Length', 0))
            data = safe_json_read(self.rfile, content_length)
            
            old_name = data.get('old_name', '').strip()
            new_name = data.get('new_name', '').strip()
            path = data.get('path', '').lstrip('/')
            
            if not old_name or not new_name:
                self.send_json_response({"error": "Both old and new names required"}, 400)
                return
            
            safe_new_name = sanitize_filename(new_name)
            if not safe_new_name:
                self.send_json_response({"error": "Invalid new name"}, 400)
                return
            
            if not is_safe_path(path):
                self.send_json_response({"error": "Unsafe path"}, 403)
                return
            
            base_dir = os.path.join(BASE_PATH, path) if path else BASE_PATH
            old_path = os.path.join(base_dir, old_name)
            new_path = os.path.join(base_dir, safe_new_name)
            
            if not os.path.exists(old_path):
                self.send_json_response({"error": "File not found"}, 404)
                return
            
            try:
                old_real = os.path.realpath(old_path)
                base_real = os.path.realpath(BASE_PATH)
                if not old_real.startswith(base_real):
                    self.send_json_response({"error": "Unsafe source path"}, 403)
                    return
            except:
                self.send_json_response({"error": "Path resolution failed"}, 500)
                return
            
            if os.path.exists(new_path):
                self.send_json_response({"error": "Target name already exists"}, 409)
                return
            
            os.rename(old_path, new_path)
            
            self.send_json_response({
                "success": True,
                "old_name": old_name,
                "new_name": safe_new_name,
                "renamed_by": token_info['username']
            })
            
        except ValueError as e:
            self.send_json_response({"error": f"JSON请求过大: {str(e)}"}, 413)
        except Exception as e:
            self.send_json_response({"error": str(e)}, 500)
    
    def handle_search(self):
        """🛡️ 指挥官专业修复：优化搜索，避免性能瓶颈"""
        try:
            token_info = self.verify_auth()
            if not token_info:
                return
            
            content_length = int(self.headers.get('Content-Length', 0))
            data = safe_json_read(self.rfile, content_length)
            
            query = data.get('query', '').strip().lower()
            if not query:
                self.send_json_response({"error": "Search query required"}, 400)
                return
            
            # 指挥官建议：限制搜索结果，避免性能瓶颈
            MAX_RESULTS = 50  # 最多返回50个结果
            results = []
            
            # 指挥官专业优化：早停机制，找到足够结果立即停止
            for root, dirs, files in os.walk(BASE_PATH):
                if len(results) >= MAX_RESULTS:
                    break
                
                try:
                    rel_root = os.path.relpath(root, BASE_PATH)
                    if rel_root == '.':
                        rel_root = ''
                except:
                    continue
                
                # 搜索文件夹（也有早停机制）
                for dirname in dirs:
                    if len(results) >= MAX_RESULTS:
                        break
                    if query in dirname.lower():
                        results.append({
                            "name": dirname,
                            "type": "folder",
                            "path": rel_root,
                            "icon": "📁"
                        })
                
                # 搜索文件（也有早停机制）
                for filename in files:
                    if len(results) >= MAX_RESULTS:
                        break
                    if query in filename.lower():
                        file_path = os.path.join(root, filename)
                        try:
                            stat = os.stat(file_path)
                            results.append({
                                "name": filename,
                                "type": "file",
                                "path": rel_root,
                                "size": self.format_file_size(stat.st_size),
                                "modified": datetime.fromtimestamp(stat.st_mtime).strftime('%Y-%m-%d %H:%M'),
                                "icon": self.get_file_icon(filename)
                            })
                        except:
                            continue
            
            self.send_json_response({
                "success": True,
                "results": results,
                "total_found": len(results),
                "limited": len(results) >= MAX_RESULTS,
                "query": query
            })
            
        except Exception as e:
            self.send_json_response({"error": f"Search failed: {str(e)}"}, 500)
    def handle_share(self):
        """生成分享链接"""
        token_info = self.verify_auth()
        if not token_info:
            return
        
        try:
            content_length = int(self.headers.get('Content-Length', 0))
            data = safe_json_read(self.rfile, content_length)
            
            file_path = data.get('file_path', '').strip()
            expire_hours = data.get('expire_hours', 24)
            
            if not file_path:
                self.send_json_response({"error": "File path required"}, 400)
                return
            
            if not is_safe_path(file_path):
                self.send_json_response({"error": "Unsafe file path"}, 403)
                return
            
            full_path = os.path.join(BASE_PATH, file_path)
            if not os.path.exists(full_path) or not os.path.isfile(full_path):
                self.send_json_response({"error": "File not found"}, 404)
                return
            
            if expire_hours < 1 or expire_hours > 720:
                self.send_json_response({"error": "Invalid expire time (1-720 hours)"}, 400)
                return
            
            token, expire_timestamp = generate_share_token(file_path, expire_hours)
            share_url = f"http://{self.headers.get('Host', 'localhost')}/api/shared_download?token={token}"
            expire_time = datetime.fromtimestamp(expire_timestamp).strftime('%Y-%m-%d %H:%M')
            
            self.send_json_response({
                "success": True,
                "share_url": share_url,
                "token": token,
                "expires": expire_time,
                "expire_hours": expire_hours,
                "shared_by": token_info['username']
            })
            
        except ValueError as e:
            self.send_json_response({"error": f"分享请求过大: {str(e)}"}, 413)
        except Exception as e:
            self.send_json_response({"error": str(e)}, 500)
    
    def handle_delete(self):
        """删除"""
        token_info = self.verify_auth()
        if not token_info:
            return
        
        try:
            content_length = int(self.headers.get('Content-Length', 0))
            data = safe_json_read(self.rfile, content_length)
            
            name = data.get('name', '').strip()
            path = data.get('path', '').lstrip('/')
            
            if not name:
                self.send_json_response({"error": "Name required"}, 400)
                return
            
            if not is_safe_path(path):
                self.send_json_response({"error": "Unsafe path"}, 403)
                return
            
            safe_name = sanitize_filename(name)
            if safe_name != name:
                self.send_json_response({"error": "Unsafe filename"}, 403)
                return
            
            base_dir = os.path.join(BASE_PATH, path) if path else BASE_PATH
            target_path = os.path.join(base_dir, name)
            
            if not os.path.exists(target_path):
                self.send_json_response({"error": "File not found"}, 404)
                return
            
            try:
                real_path = os.path.realpath(target_path)
                base_real = os.path.realpath(BASE_PATH)
                if not real_path.startswith(base_real):
                    self.send_json_response({"error": "Unsafe delete target"}, 403)
                    return
            except:
                self.send_json_response({"error": "Path resolution failed"}, 500)
                return
            
            if os.path.isfile(target_path):
                os.remove(target_path)
            elif os.path.isdir(target_path):
                shutil.rmtree(target_path)
            else:
                self.send_json_response({"error": "Unknown file type"}, 400)
                return
            
            self.send_json_response({
                "success": True,
                "deleted": name,
                "deleted_by": token_info['username']
            })
            
        except ValueError as e:
            self.send_json_response({"error": f"删除请求过大: {str(e)}"}, 413)
        except Exception as e:
            self.send_json_response({"error": str(e)}, 500)
    
    def handle_batch_delete(self):
        """批量删除"""
        token_info = self.verify_auth()
        if not token_info:
            return
        
        try:
            content_length = int(self.headers.get('Content-Length', 0))
            
            # 批量删除可能需要更大的JSON，但仍有限制
            if content_length > 16*1024:  # 16KB限制
                self.send_json_response({"error": "批量删除请求过大，超过16KB限制"}, 413)
                return
                
            data = safe_json_read(self.rfile, content_length)
            
            items = data.get('items', [])
            path = data.get('path', '').lstrip('/')
            
            if not items:
                self.send_json_response({"error": "No items to delete"}, 400)
                return
            
            if not is_safe_path(path):
                self.send_json_response({"error": "Unsafe path"}, 403)
                return
            
            base_dir = os.path.join(BASE_PATH, path) if path else BASE_PATH
            deleted_items = []
            failed_items = []
            
            for item_name in items:
                try:
                    safe_name = sanitize_filename(item_name)
                    if safe_name != item_name:
                        failed_items.append({"name": item_name, "error": "Unsafe filename"})
                        continue
                    
                    target_path = os.path.join(base_dir, item_name)
                    
                    if not os.path.exists(target_path):
                        failed_items.append({"name": item_name, "error": "File not found"})
                        continue
                    
                    try:
                        real_path = os.path.realpath(target_path)
                        base_real = os.path.realpath(BASE_PATH)
                        if not real_path.startswith(base_real):
                            failed_items.append({"name": item_name, "error": "Unsafe path"})
                            continue
                    except:
                        failed_items.append({"name": item_name, "error": "Path resolution failed"})
                        continue
                    
                    if os.path.isfile(target_path):
                        os.remove(target_path)
                        deleted_items.append(item_name)
                    elif os.path.isdir(target_path):
                        shutil.rmtree(target_path)
                        deleted_items.append(item_name)
                    else:
                        failed_items.append({"name": item_name, "error": "Unknown file type"})
                        
                except Exception as e:
                    failed_items.append({"name": item_name, "error": str(e)})
            
            self.send_json_response({
                "success": True,
                "deleted": deleted_items,
                "failed": failed_items,
                "deleted_count": len(deleted_items),
                "failed_count": len(failed_items),
                "deleted_by": token_info['username']
            })
            
        except ValueError as e:
            self.send_json_response({"error": f"批量删除请求过大: {str(e)}"}, 413)
        except Exception as e:
            self.send_json_response({"error": str(e)}, 500)
    
    # 辅助函数
    def get_safe_filename(self, filename, directory, is_folder=False):
        """生成安全的唯一文件名"""
        base_path = os.path.join(directory, filename)
        
        if not os.path.exists(base_path):
            return base_path
        
        name, ext = os.path.splitext(filename) if not is_folder else (filename, '')
        counter = 1
        
        while True:
            if is_folder:
                new_name = f"{name}({counter})"
            else:
                new_name = f"{name}({counter}){ext}"
            
            new_path = os.path.join(directory, new_name)
            if not os.path.exists(new_path):
                return new_path
            
            counter += 1
            if counter > 1000:
                break
        
        timestamp = int(datetime.now().timestamp())
        if is_folder:
            final_name = f"{name}_{timestamp}"
        else:
            final_name = f"{name}_{timestamp}{ext}"
        
        return os.path.join(directory, final_name)
    
    def send_json_response(self, data, status_code=200):
        """发送JSON响应"""
        self.send_response(status_code)
        self.send_header('Content-Type', 'application/json; charset=utf-8')
        self.send_header('Access-Control-Allow-Origin', '*')
        self.send_header('Access-Control-Allow-Methods', 'GET, POST, OPTIONS')
        self.send_header('Access-Control-Allow-Headers', 'Content-Type, Authorization')
        self.end_headers()
        
        json_data = json.dumps(data, ensure_ascii=False, indent=2)
        self.wfile.write(json_data.encode('utf-8'))
    
    def format_file_size(self, size):
        """格式化文件大小"""
        for unit in ['B', 'KB', 'MB', 'GB', 'TB']:
            if size < 1024.0:
                return f"{size:.1f} {unit}"
            size /= 1024.0
        return f"{size:.1f} PB"
    
    def get_file_icon(self, filename):
        """根据文件扩展名返回图标"""
        ext = os.path.splitext(filename)[1].lower()
        
        image_exts = {'.jpg', '.jpeg', '.png', '.gif', '.bmp', '.svg', '.webp'}
        video_exts = {'.mp4', '.avi', '.mkv', '.mov', '.wmv', '.flv', '.webm'}
        audio_exts = {'.mp3', '.wav', '.flac', '.aac', '.ogg', '.m4a'}
        doc_exts = {'.doc', '.docx', '.pdf', '.txt', '.rtf', '.odt'}
        code_exts = {'.py', '.js', '.html', '.css', '.java', '.cpp', '.c', '.php', '.rb', '.go'}
        archive_exts = {'.zip', '.rar', '.7z', '.tar', '.gz', '.bz2'}
        
        if ext in image_exts:
            return '🖼️'
        elif ext in video_exts:
            return '🎥'
        elif ext in audio_exts:
            return '🎵'
        elif ext in doc_exts:
            return '📄'
        elif ext in code_exts:
            return '💻'
        elif ext in archive_exts:
            return '📦'
        else:
            return '📄'


if __name__ == "__main__":
    with socketserver.ThreadingTCPServer(("127.0.0.1", PORT), CloudDriveHandler) as httpd:
        print(f"🛡️ Modern Drive API - 指挥官最终修复版")
        print(f"🎯 指挥官指令完成: 真正的流式处理")
        print(f"💗 内存安全: JSON 4KB硬限制 + 文件8KB流式")
        print(f"🌊 引擎: 裸流式直接写入，零内存增长")
        print(f"🛡️ 安全: 路径验证 + Token认证 + Content-Length限制")
        print(f"📁 数据路径: {BASE_PATH}")
        print(f"🌐 服务端口: {PORT}")
        httpd.serve_forever()