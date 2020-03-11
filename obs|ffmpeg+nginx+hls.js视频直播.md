## 视频直播
本方案采用obs[ffmpeg]采集或是推流到nginx，然后通过hls.js来拉流。
涉及的软件地址如下：
 + obs https://obsproject.com/
 + denji/nginx
 + https://github.com/video-dev/hls.js
 + ffmpeg
 + 操作系统 mac

## nginx安装

```shell
brew tap denji/nginx
brew install nginx-full --with-rtmp-module
brew info nginx-full
nginx 启动
```

配置文件
```conf
http的server节点增加
  location /hls {
    # 响应类型
      types {
        application/vnd.apple.mpegurl m3u8;
        video/mp2t ts;
      }
      root /usr/local/var/www;
      # 不要缓存
      add_header Cache-Control no-cache;
      autoindex on;
  
  expires -1;
  add_header Cache-Control no-cache;
  #防止跨域问题
  add_header 'Access-Control-Allow-Origin' '*';
  add_header 'Access-Control-Allow-Credentials' 'true';
  add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
  add_header 'Access-Control-Allow-Headers' 'DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type';  
  
http平行节点增加

rtmp {
    server {
      # 监听端口
      listen 1935;
      # 分块大小
      chunk_size 4000;

      # RTMP 直播流配置
      application rtmplive {
        # 开启直播的模式
        live on;
        # 设置最大连接数
        max_connections 1024;
      }

      # hls 直播流配置
      application hls {
        live on;
        hls on;
        # 分割文件的存储位置
        hls_path /usr/local/var/www/hls;
        # hls分片大小
        hls_fragment 5s;
      }
    }
}
```

nginx -s reload

## hls
下载hls，地址见开头
```
npm run dev
```
浏览器打开 http://localhost:8000/ 进demo目录

```
你可以用ffmpeg将大视频拆分成小块，在本地验证hls（见文件夹放到demo文件夹，选择hls指定M3U8D地址即可）
ffmpeg -i outputvideo.ts  -c copy -map 0 -f segment -segment_list playlist.m3u8 -segment_time 10  video_%03d.ts
```

## OBS安装
下载obs，地址见开头。安装

## 联调
在obs软件中source中增加Video Capture Device
在settings中的stream中增加
source：custom
server：rtmp://localhost:1935/hls
streamkey:stream {随意指定，该名字是M3U8的名字}

在hls中选择hls.js/issues/666
修改M3U8地址为：http://localhost:8080/hls/stream.m3u8

（测试本地ffmpeg切分的视频地址为：http://localhost:8000/demo/data/playlist.m3u8【假设将文件放到data文件夹】）

## 最后
本文更多的是流水账，亲测可行。将ts、M3U8文件放到CDN来加速播放由于当前环境限制暂没有体验。后续希望有机会补充。