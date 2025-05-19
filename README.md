# box.com-shared-drive-link-folder-downloader
box.com 网盘分享文件夹，链接下载，python脚本



```
#!/usr/bin/env python3
import sys
import json
import re
import os
import requests
import time
from urllib.parse import unquote, quote
from datetime import datetime

# 代理服务器配置
PROXY_SERVER = "https://c.map987.dpdns.org/"

def format_size(bytes_size):
    """格式化文件大小"""
    for unit in ['B', 'KB', 'MB', 'GB']:
        if bytes_size < 1024.0:
            return f"{bytes_size:.2f} {unit}"
        bytes_size /= 1024.0
    return f"{bytes_size:.2f} TB"

def extract_folder_id_and_size(html_content):
    """
    从HTML内容提取文件夹ID和文件大小信息
    返回: (folder_id, total_size) 元组
    """
    # 查找包含postStreamData的行
    lines = [line for line in html_content.split('\n') if 'postStreamData' in line]
    if not lines:
        print("错误：未找到postStreamData内容")
        return None, None
    
    print("找到关键行（前200字符）:\n", lines[0][:200], "...")
    print("html中的这行是json格式，在一个script标签内↑:")
    
    # 提取folder对象
    folder_match = re.search(r',"folder":({[^}]*})', lines[0])
    if not folder_match:
        print("错误：无法提取folder对象")
        return None, None
    
    # 提取ID
    id_match = re.search(r'"id":([^,]+)', folder_match.group(1))
    if not id_match:
        print("错误：无法从folder对象提取ID")
        return None, None
    folder_id = id_match.group(1).strip('"\' ')
    
    # 提取文件总大小（从items数组中的各个文件大小汇总）
    size_match = re.search(r'"items":\[({.*?})\]', lines[0], re.DOTALL)
    total_size = 0
    if size_match:
        # 提取所有itemSize字段并求和
        size_matches = re.finditer(r'"itemSize":(\d+)', size_match.group(1))
        total_size = sum(int(m.group(1)) for m in size_matches)
    
    return folder_id, total_size

def download_with_progress(url, file_path, total_size=0, proxy=False):
    """带单行刷新进度显示的下载函数"""
    start_time = time.time()
    downloaded = 0
    
    # 设置代理URL
    final_url = f"{PROXY_SERVER}{quote(url, safe='')}" if proxy else url
    
    try:
        response = requests.get(final_url, stream=True, timeout=60)
        response.raise_for_status()
        
        # 如果未传入total_size，尝试从headers获取
        if total_size <= 0:
            total_size = int(response.headers.get('content-length', 0))
        
        file_size_str = format_size(total_size) if total_size > 0 else "未知大小"
        
        # 初始化进度显示
        sys.stdout.write("\r下载进度: 0.0% | 0.00 B/{} | 速度: 0.00 B/s".format(file_size_str))
        sys.stdout.flush()
        
        with open(file_path, 'wb') as f:
            for chunk in response.iter_content(chunk_size=8192):
                if chunk:  # 过滤保持连接的chunk
                    f.write(chunk)
                    downloaded += len(chunk)
                    
                    # 计算进度
                    elapsed = time.time() - start_time
                    speed = downloaded / elapsed if elapsed > 0 else 0
                    percent = downloaded / total_size * 100 if total_size > 0 else 0
                    
                    # 更新进度显示
                    sys.stdout.write(
                        "\r下载进度: {:6.1f}% | {}/{} | 速度: {}/s".format(
                            percent,
                            format_size(downloaded),
                            file_size_str,
                            format_size(speed)
                        )
                    )
                    sys.stdout.flush()
        
        # 下载完成后清除进度行，显示完成信息
        sys.stdout.write("\r" + " " * 80 + "\r")
        print(f"下载完成: {format_size(downloaded)} 用时: {time.time()-start_time:.1f}秒")
        return True
    
    except Exception as e:
        sys.stdout.write("\r" + " " * 80 + "\r")
        print(f"下载失败: {str(e)}")
        return False

def download_box_file(url, target_dir="."):
    """带完整代理支持的主下载函数"""
    html_file = None
    json_file = None
    
    try:
        # 确保目标目录存在
        os.makedirs(target_dir, exist_ok=True)
        
        # 获取分享ID
        share_id = url.split('/')[-1]
        html_file = os.path.join(target_dir, f"{share_id}.html")
        
        # 通过代理下载HTML页面
        proxy_url = f"{PROXY_SERVER}{quote(url, safe='')}"
        print(f"[{datetime.now().strftime('%H:%M:%S')}] 正在通过代理下载页面: {proxy_url}")
        
        if not download_with_progress(url, html_file, proxy=True):
            raise ValueError("页面下载失败")
        
        # 提取文件夹ID和文件大小
        with open(html_file, 'r', encoding='utf-8') as f:
            html_content = f.read()
        
        folder_id, total_size = extract_folder_id_and_size(html_content)
        if not folder_id:
            raise ValueError("无法提取文件夹ID")
        
        print(f"[{datetime.now().strftime('%H:%M:%S')}] 成功获取文件夹ID: {folder_id}")
        if total_size > 0:
            print(f"[{datetime.now().strftime('%H:%M:%S')}] 预估文件总大小: {format_size(total_size)}")
        
        # 通过代理获取下载信息
        api_path = (
            f"https://app.box.com/index.php?"
            f"folder_id={folder_id}&"
            f"q%5Bshared_item%5D%5Bshared_name%5D={share_id}&"
            "rm=box_v2_zip_shared_folder"
        )
        proxy_api_url = f"{PROXY_SERVER}{quote(api_path, safe='')}"
        
        json_file = os.path.join(target_dir, f"{folder_id}.json")
        print(f"[{datetime.now().strftime('%H:%M:%S')}] 正在获取下载信息: {proxy_api_url}")
        
        response = requests.get(proxy_api_url, timeout=30)
        response.raise_for_status()
        json_data = response.json()
        
        print("\n============携带下载链接的下载信息json已经获取到：")
        print(json.dumps(json_data, indent=2))
        
        download_url = json_data.get('download_url')
        if not download_url:
            raise ValueError("API响应中没有download_url字段")
        
        print(f"[{datetime.now().strftime('%H:%M:%S')}] 获取到下载链接: {download_url[:50]}...")
        
        # 提取文件名
        file_name_match = re.search(r'ZipFileName=([^&]+)', download_url)
        if file_name_match:
            file_name = unquote(file_name_match.group(1))
        else:
            file_name = f"box_download_{folder_id}.zip"
        
        output_path = os.path.join(target_dir, file_name)
        
        # 下载最终文件
        print(f"[{datetime.now().strftime('%H:%M:%S')}] 开始下载文件: {file_name}")
        
        """
        注： 前两个requests请求，包括:
        (1)下载网盘分享链接的html，
        (2)和从html获取文件夹id后，使用 https://app.box.com/index.php?folder_id={folder_id}&q%5Bshared_item%5D%5Bshared_name%5D={share_id}&rm=box_v2_zip_shared_folder 去获取最终下载链接，
        上述两个box.com，都无法直连，
        ===========        

        下面↓这个requests请求，就是最终下载链接 https://dl.boxcloud.com/………
        这个可以直接连接但是下载速度很慢
        """
        
        # 使用带进度显示的下载函数
        if not download_with_progress(download_url, output_path, total_size=total_size):
            raise ValueError("文件下载失败")
        
        print(f"[{datetime.now().strftime('%H:%M:%S')}] ✅ 下载完成: {output_path}")
        return True
        
    except requests.exceptions.RequestException as e:
        print(f"[{datetime.now().strftime('%H:%M:%S')}] 网络请求失败: {str(e)}")
    except json.JSONDecodeError as e:
        print(f"[{datetime.now().strftime('%H:%M:%S')}] JSON解析失败: {str(e)}")
    except Exception as e:
        print(f"[{datetime.now().strftime('%H:%M:%S')}] 发生错误: {str(e)}")
    finally:
        # 清理临时文件
        for temp_file in [f for f in [html_file, json_file] if f and os.path.exists(f)]:
            try:
                os.remove(temp_file)
                print(f"[{datetime.now().strftime('%H:%M:%S')}] 已清理临时文件: {temp_file}")
            except:
                pass
    
    return False

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("使用方法: python box_downloader.py <box_url> [目标目录]")
        print("示例: python box_downloader.py https://app.box.com/s/abc123 ./downloads")
        sys.exit(1)
    
    box_url = sys.argv[1]
    target_dir = sys.argv[2] if len(sys.argv) > 2 else "."
    
    if not download_box_file(box_url, target_dir):
        sys.exit(1)
```

proxy为
cloudflare workers 脚本:
`https://xx.workers.dev/https://sample.com/1.jpg` = `https://sample.com/1.jpg`
