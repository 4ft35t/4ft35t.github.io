---
title: "群晖 NAS 上运行 PaddleOCR"
description: 在群晖 NAS 上折腾百度 PaddleOCR 的踩坑过程
date: 2022-04-21T19:22:21+08:00
image: https://raw.githubusercontent.com/PaddlePaddle/PaddleOCR/release/2.6/doc/PaddleOCR_log.png
math: 
hidden: false
comments: true
tags: [docker, linux, ocr, error, python, paddleocr]
categories: [Linux]
draft: false
---

本文的起因是 Manjaro 的 python 版本升级到 3.10 后，PaddleOCR 无法运行。尝试用 pyenv 安装 3.9.10 版 python 后，执行命令 `paddleocr` 报错
> ImportError: /opt/github/ocr/venv3910/lib/python3.9/site-packages/paddle/fluid/core_avx.so: undefined symbol: _dl_sym, version GLIBC_PRIVATE

这个报错，在 PaddleOCR 官方的 github 仓库 issue 有人发过, ubuntu 20.10 上遇到此报错，https://github.com/PaddlePaddle/Paddle/issues/36801 。

只能把 PaddleOCR 放在 Docker 运行了。不过官方镜像的体积高达 11G，我只需要 OCR 功能，不做样本训练，应该有很大的精简空间。

此前在 VPS 上的 ubuntu 20.04 里用过 PaddleOCR，故选择 ubuntu 20.04 做基础镜像。群晖 NAS 上有多个 Docker 服务，PaddleOCR 同样丢群晖跑。

## 运行 ubuntu 20.04
ssh 登录群晖后，执行 `sudo su`，输入当前用户的密码，切换到 __root__。

执行下面命令，进入 ubuntu 的 bash 终端
```
docker pull ubuntu:20.04
docker run -it ubuntu:20.04 bash
```

## 系统的软件源换成阿里云
```
cp /etc/apt/sources.list{,.bak}
sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/' /etc/apt/sources.list
apt-get update
```

## 安装 python 环境
```
apt-get install python3 python3-pip wget
mkdir /root/.pip
echo '[global]' > /root/.pip/pip.conf
echo 'index-url = https://pypi.doubanio.com/simple' >> /root/.pip/pip.conf
```

## 安装 PaddleOCR
按官方文档操作，https://github.com/PaddlePaddle/PaddleOCR/blob/release/2.4/doc/doc_ch/quickstart.md
```
pip install paddlepaddle -i https://mirror.baidu.com/pypi/simple
pip install paddleocr
```
一切进展顺利，下面开始踩坑

## 填坑过程

执行 `paddleocr -h`, 看是否有报错信息。错误一个接一个的来，下面安出现顺序解决

### ModuleNotFoundError: No module named 'paddle.fluid.core_noavx'

群晖的 CPU 太古老了，不支持 AVX(Advanced Vector Extensions) 指令集, 需要 noavx 的预编译包。
在这个 issue https://github.com/PaddlePaddle/PaddleOCR/issues/4275 中找到线索，官方的预先编译 whl 列表 https://www.paddlepaddle.org.cn/documentation/docs/zh/develop/install/Tables.html#whl-release

`cpu-mkl-noavx` 版本的预编译 whl 只有 python 3.8 的，正好和 ubuntu 20.04 的 python 版本一致。

下载地址

> https://paddle-wheel.bj.bcebos.com/2.2.1/linux/linux-cpu-mkl-noavx/paddlepaddle-2.2.1-cp38-cp38-linux_x86_64.whl

当前版本已是 2.2.2, 把连接中 2.2.1 全部替换成 2.2.2

```
echo https://paddle-wheel.bj.bcebos.com/2.2.1/linux/linux-cpu-mkl-noavx/paddlepaddle-2.2.1-cp38-cp38-linux_x86_64.whl | sed 's/2.2.1/2.2.2/g'
```

下载安装, 此问题解决

```
wget https://paddle-wheel.bj.bcebos.com/2.2.2/linux/linux-cpu-mkl-noavx/paddlepaddle-2.2.2-cp38-cp38-linux_x86_64.whl
pip install --force-reinstal paddlepaddle-2.2.2-cp38-cp38-linux_x86_64.whl
```

### ImportError: libGL.so.1: cannot open shared object file: No such file or directory

`pip install opencv-python-headless==4.2.0.32`

### ImportError: libgthread-2.0.so.0: cannot open shared object file: No such file or directory

`apt-get install libglib2.0-0 libsm6 libxrender1 libxext6`

中途会出现选择时区的交互，依次选择 `6. Asia` - `19. Chongqing`

### ImportError: libgomp.so.1: cannot open shared object file: No such file or directory
`apt-get install libgomp1`

至此，paddleocr 可以正常运行。

## 添加 API
使用 flask 提供简单的 ocr 识别 API，可以上传图片或者从 url 下载图片，输出格式可选 text 或者 json，默认 text。

```python
@app.route('/ocr', methods=['PUT', 'POST'])
def ocr_text():
    img = request.files.get('img')
    if not img:
        img_url = request.form['imgurl']
        img = requests.get(img_url).content
        img = BytesIO(img)
    else:
        file_obj = BytesIO()
        img.save(file_obj)
        img = file_obj
    img = Image.open(img).convert('RGB')

    result = ocr.ocr(np.array(img), det=True, rec=True, cls=True)
    text = '\n'.join([i[1][0] for i in result])

    output_format = request.form.get('outtype', 'text')
    if output_format == 'text':
        return text
    ret = {'success': True, 'results': []}
    for each in result:
        item = {
            'confidence': each[1][1].item(),
            'text': each[1][0],
            'text_region': each[0]
            }
        ret['results'].append(item)
    return ret
```

### API 使用
- 上传文件识别, 输出 text 格式

  ` curl 127.0.0.1:5000/ocr -F img=@/tmp/img.png`

- 从图片url识别, 并输出 json 格式
  `curl 127.0.0.1:5000/ocr -F imgurl=https://raw.githubusercontent.com/PaddlePaddle/PaddleOCR/release/2.4/doc/imgs_en/254.jpg -F outtype=json`

## 创建 Dockerfile
上面提到的安装过程中需要交互选择时区，会中断 docker 编译，通过设置 `ENV DEBIAN_FRONTEND=noninteractive` 来规避
```
FROM ubuntu:20.04

EXPOSE 5000
ENV TZ=Asia/Shanghai
ENV DEBIAN_FRONTEND=noninteractive

ARG PADDLE_VER=2.2.2

COPY app.py /
COPY requirements.txt /tmp

RUN apt-get update \
    && apt-get install -y python3 python3-pip libgomp1 libglib2.0-0 libsm6 libxrender1 libxext6 \
    && pip install -r /tmp/requirements.txt \
    && wget --directory-prefix /tmp/ https://paddle-wheel.bj.bcebos.com/${PADDLE_VER}/linux/linux-cpu-mkl-noavx/paddlepaddle-${PADDLE_VER}-cp38-cp38-linux_x86_64.whl \
    && pip install --force-reinstall /tmp/paddlepaddle-${PADDLE_VER}-cp38-cp38-linux_x86_64.whl
RUN apt-get -y remove python3-pip \
    && apt-get -y autoremove \
    && apt-get -y install --no-install-recommends python3-setuptools \
    && rm -rf /var/lib/{apt,dpkg,cache,log}/ /root/.cache /tmp/*

CMD ["python3", "/app.py"]
```

Dockerfile 及其它文件在 Github Repo https://github.com/4ft35t/Paddleocr-docker

Docker Hub 地址 https://hub.docker.com/r/c403/paddleocr

最后吐槽一下，PaddleOCR 的文档太混乱了, 很多内容在 issues 里，不在文档。

## 参考链接
- https://github.com/PaddlePaddle/PaddleOCR/issues/4275
- https://www.paddlepaddle.org.cn/documentation/docs/zh/develop/install/Tables.html#whl-release
- https://blog.csdn.net/Max_ZhangJF/article/details/108920050
- https://github.com/PaddlePaddle/PaddleOCR/issues/1735
- https://stackoverflow.com/questions/62786028/importerror-libgthread-2-0-so-0-cannot-open-shared-object-file-no-such-file-o
- https://github.com/explosion/spaCy/issues/1110
- https://askubuntu.com/questions/909277/avoiding-user-interaction-with-tzdata-when-installing-cert-in-a-docker-contai
