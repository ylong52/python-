/*

这段Python代码是一个简单的脚本，用于从RTMP流中拉取视频并将其分割成多个MP4分段文件。它还使用RabbitMQ将当前时间发送到指定的队列。以下是代码的详细说明：

导入所需的库和模块：
os：用于操作系统的命令行界面。
re：用于正则表达式操作。
subprocess：用于执行子进程。
pika：用于RabbitMQ通信。
time：用于时间操作。
ffmpeg：用于FFmpeg命令行工具的Python封装。
定义常量：
RABBITMQ_HOST：RabbitMQ服务器的地址。
current_time：当前时间的字符串格式。
file_name_pattern：分段文件的命名模式，包括时间戳和序号。
segment_files：存储分段文件名的列表。
创建分段文件： 代码创建了一个列表segment_files，其中包含了1000个分段文件名。这些文件名是基于当前时间和序号生成的，例如stream_piece_20230101123456_001.mp4。
FFmpeg命令行参数： 这部分定义了ffmpeg的命令行参数，用于从RTMP流中拉取视频并将其分割成MP4分段文件。这些参数包括输入流、输出格式、分段时间、视频尺寸等。
RabbitMQSender函数： 这个函数用于与RabbitMQ服务器通信。它建立一个到指定RabbitMQ服务器的连接，声明一个名为hello的队列，然后发送当前时间的字符串到这个队列。
start_streaming函数： 这个函数开始拉取RTMP流并分割文件。它使用了ffmpeg-python库来执行ffmpeg命令。ffmpeg命令将RTMP流转换为MP4分段文件，并将分段列表写入segment_list.txt文件。
主函数： 如果脚本作为主程序运行，它会调用start_streaming函数来开始拉取和分割RTMP流，然后调用RabbitMQSender函数来发送当前时间到RabbitMQ服务器。
在将代码上传到Git时，请确保遵循以下最佳实践：

添加适当的注释和文档，以便其他人能够理解代码的功能和用途。
确保所有的变量和函数都有明确的用途和说明。
使用版本控制软件（如Git）来管理代码的变更。
在代码中使用适当的命名约定和格式，以便代码易于阅读和维护


*/

import os
import re
import subprocess

import pika
import time
from datetime import datetime

import ffmpeg

#定义常量
RABBITMQ_HOST = '82.156.207.54'
current_time = time.strftime('%Y%m%d%H%M%S')
file_name_pattern = f'stream_piece_{current_time}_%d.mp4'
# 创建分段文件
segment_files = []
for i in range(1000):  # 假设你想要创建 100 个分段
    segment_file = f'{current_time}_{i:03d}.mp4'
    segment_files.append(segment_file)

# FFmpeg命令行参数
ffmpeg_cmd = [
    # 指定FFmpeg的可执行文件路径（如果不在系统路径中）
    'ffmpeg',
    # 指定输入文件，这里使用RTMP流
    '-i', 'rtmp://82.156.207.54/live/livestream',
    # 指定输出文件格式为MP4，并使用复制（copy）编码方式
    '-c', 'copy',
    # 指定输出格式为分段（segment）
    '-f', 'segment',
    # 指定每个分段的时长为60秒
    '-segment_time', '5',
    '-s', '320x240',  # 设置视频尺寸为320x240
    # 指定分段格式为MP4
    '-segment_format', 'mp4',
    # 指定分段文件名格式为时间戳
     '-strftime', f'{current_time}_%d.mp4'
    # 输出分段文件列表
    '-write_playlist', '1',

]

rtmp_url = "rtmp://82.156.207.54/live/livestream"
# ffmpeg_cmd2 = f'ffmpeg -i {rtmp_url} -c copy -f segment -segment_time 60 stream_piece_%%s.mp4'



# 分阶压片的函数
# https://kkroening.github.io/ffmpeg-python/
# 开始拉流并分割文件
def start_streaming():

    current_time = time.strftime('%Y%m%d%H%M%S')
    file_name = f'stream_piece_{current_time}_%d.mp4'
    ffmpeg_cmd2 = f'ffmpeg -i {rtmp_url} -c copy -f segment -segment_list segment_list.txt -segment_time 5  {file_name}'

    process = (
        ffmpeg
            .input(rtmp_url)
            .output(file_name, **{'format': 'segment', 'segment_list': 'segment_list.txt', 'segment_time': 5})
            .run_async()
    )

    out, err = process.communicate()


# RabbitMQSender队列发送代码
def RabbitMQSender():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters(host=RABBITMQ_HOST))
    channel = connection.channel()

    channel.queue_declare(queue='hello',durable=True)

    current_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    # print(current_time)
    channel.basic_publish(exchange='',
                          routing_key='hello',
                          body=current_time,
                          properties=pika.BasicProperties(
                             delivery_mode = 2,  # make message persistent
                          ))

    print(f" [x] Sent  {current_time}")
    connection.close()



if __name__ == '__main__':
    start_streaming()

