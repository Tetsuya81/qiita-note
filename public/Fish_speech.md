---
title: Fish speechを使ってみる
tags:
  - 'Fish_speech, Ubuntu'
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
# Fish speechとは？
Fish-Speechは、FishAudioによって開発された最新のテキスト音声合成（TTS）ソリューションです。このプロジェクトは、GitHub上で公開されており、コミュニティからの貢献を受け入れています。

![Fish speech: Github](https://github.com/fishaudio/fish-speech)

# 導入環境
- Ubuntu 24.04
  - ハード
    - Intel i5 7500
    - RTX 4060 8GB
  - 事前に必要なもの
    - [Personal — conda-forge/miniforge: A conda-forge distribution.](https://github.com/conda-forge/miniforge)


# 導入方法


```bash
git clone https://github.com/fishaudio/fish-speech.git
cd fish-speech
```

[Personal — Fish Speech の紹介 - Fish Speech](https://speech.fish.audio/ja/)
```bash
# python 3.10の仮想環境を作成します。virtualenvも使用できます。
conda create -n fish-speech python=3.10
conda activate fish-speech

# pytorchをインストールします。
pip3 install torch==2.4.1 torchvision==0.19.1 torchaudio==2.4.1

# (Ubuntu / Debianユーザー) sox + ffmpegをインストールします。
apt install libsox-dev ffmpeg

# (Ubuntu / Debianユーザー) pyaudio をインストールします。
apt install build-essential \
    cmake \
    libasound-dev \
    portaudio19-dev \
    libportaudio2 \
    libportaudiocpp0

# fish-speechをインストールします。
pip3 install -e .[stable]
```

## Dockerのセットアップ

### NvidiaのGPUを使用する場合
```bash
# リポジトリの追加
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
    && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
        sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
        sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
# nvidia-container-toolkit のインストール
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
# Docker サービスの再起動
sudo systemctl restart docker
```

### Dockerイメージを実行

```bash
# イメージのプル
docker pull fishaudio/fish-speech:latest-dev
# イメージの実行
docker run -it \
    --name fish-speech \
    --gpus all \
    -p 7860:7860 \
    fishaudio/fish-speech:latest-dev \
    zsh
# 他のポートを使用する場合は、-p パラメータを YourPort:7860 に変更してください
```

*補足: dockerコマンドの権限がなかった場合*
```bash
// docekrグループに現在のユーザーを追加
sudo usermod -aG docker [ユーザー名]

// 変更内容を反映させる
newgrp docker
```

### Dockerを再起動する
```
docker start -ai fish-speech
```

# Dockerの構築でエラーが発生する場合
今回、Fish-speechの環境を構築するにあたって、以下のエラーが発生しました。

## CUDAのバージョン不一致
Ubuntu 24.04で使用するインストール済みのCUDAをUbuntu22.04用でインストールしていた。

OS, CUDAのバージョン確認方法
```bash
echo "🧩 OS:"
lsb_release -a

echo -e "\n🖥️ NVIDIA Driver & CUDA:"
nvidia-smi

echo -e "\n🔧 CUDA Toolkit (nvcc):"
nvcc --version || echo "nvcc not found"
```

下記のURLからOSのバージョンにあったCUDAを選んで、手順に沿ってインストールする。
![Personal — CUDA Toolkit - Free Tools and Training | NVIDIA Developer](https://developer.nvidia.com/cuda-toolkit)

## Dockerの重複によるエラー
Dockerがapt-getとsnapの両方でインストールされていため、Dockerでイメージを構築する際にNvidiaのライブラリが読み込まれない
```bash
nvidia-container-cli: initialization error: load library failed: libnvidia-ml.so.1: cannot open 
```

### dockerをインストールした方法を確認

```bash
which docker
dpkg -l | grep docker
snap list | grep docker
```

- `/usr/bin/docker` なら APT でのインストールの可能性が高い。
- `/snap/bin/docker` なら Snap でのインストール。
- `/usr/local/bin/docker` ならバイナリを手動で配置した可能性が高い。

確認の結果、APTとSnapで確認ができた。
APTを残すため、Snapでインストールしたdockerを削除する。

### Snapでインストールしたdockerを削除

不要なコンテナ、ボリュームを停止、削除する
```bash
docker ps -a
docker stop <container_id>
docker rm <container_id>
docker volume ls
docker volume rm <volume_name>
```

snapで起動していたdockerを停止する
```bash
sudo systemctl stop snap.docker.dockerd.service
sudo systemctl disable snap.docker.dockerd.service
```

Snapのdockerを削除する。(自動でスナップショットが取られるため、強制的に削除した)
```bash
sudo snap remove --purge docker
```

### dockerをクリーンインストールする

```bash
# 1. Docker関連を削除
sudo apt remove containerd containerd.io docker docker.io docker-compose

# 2. 依存パッケージのインストール
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# 3. Docker公式GPGキーを追加
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --yes --dearmor -o /etc/apt/keyrings/docker.gpg

# 4. リポジトリを追加（Ubuntu 24.04 Noble用）
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu noble stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 5. パッケージリスト更新
sudo apt update

# 6. Docker本体のインストール
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

dockerの起動確認
```bash
sudo systemctl enable --now docker
sudo docker run hello-world
```