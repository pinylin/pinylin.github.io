---
layout: post
title: Ollama Pitfull Notes
subtitle: Ollama踩坑记录
author: pinylin
header-style: text
catalog: true
tags:
  - LLM
  - Ollama
---

## 1. Open-WebUI could not connect to Ollama

**If Ollama is on your computer**, use this command:
```sh
docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```

首先，根据官方文档, 上面这个run 起来之后，Ollama connection http://localhost:11434 是连不上的
其次，你根据提示在上面的基础上加了  --network=host 也是不行的，-p 3000:8080 有问题
所以，最后使用下边的命令， 直接 http://localhost:8080

```
docker run -d --network=host -v open-webui:/app/backend/data -e OLLAMA_API_BASE_URL=http://127.0.0.1:11434/api --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```

## 2. Ollama use CPU, not GPU

用llama3:8b的时候没发现，llama3:70b时发现CPU狂飙几min GPU没动静，所以发现：

- use GPU:  ollama run llama3
- use CPU:  ollama serve

1. 首先排查 CUDA_VISIBLE_DEVICES， 结果只有一张显卡(废话)
 ```sh
 lspci | grep VGA	
 ```
1. rocm-sdk： 因为是Ryzen 5800h
```
pacman -S rocm-hip-sdk rocm-opencl-sdk clblast go
```

效果是很明显的，
- llama3:8b能非常好的使用GPU (50W+)
- llama3:70b GPU 从10W 到 20W左右， 这个应该跟rocm有关系吧，，，只能等待优化了

```
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 3060 ...    Off |   00000000:01:00.0  On |                  N/A |
| N/A   36C    P0             22W /   80W |    5777MiB /   6144MiB |     13%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
```