## Ollama

### Шаг 1 — установить Ollama:
```
curl -fsSL https://ollama.com/install.sh | sh
```

### Шаг 2 — настроить Ollama чтобы слушал все интерфейсы (нужно для Docker):

```
sudo mkdir -p /etc/systemd/system/ollama.service.d/

sudo tee /etc/systemd/system/ollama.service.d/override.conf << 'EOF'
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_KEEP_ALIVE=-1"
EOF

sudo systemctl daemon-reload
sudo systemctl restart ollama
```
**Проверка:**
```
ss -tlnp | grep 11434
```

### Шаг 3 — загрузить модели из готовых GGUF файлов:

```
# Создать папку для Modelfile конфигов
mkdir -p ~/ai_infratest/modelfiles

# По желанию редактируем системные промпты
cat > ~/ai_infratest/modelfiles/qwen-coder-3b << 'EOF'
FROM /home/tnv/ai_infratest/models/qwen2.5-coder-3b-instruct-q4_k_m.gguf
SYSTEM "You are an expert embedded Linux developer specializing in C programming, Linux kernel drivers, RTOS, HiSilicon hi3518 and Rockchip SoCs. Always write clean efficient C code with error handling and comments."
PARAMETER num_gpu 99
PARAMETER num_ctx 2048
EOF

cat > ~/ai_infratest/modelfiles/qwen-coder-7b << 'EOF'
FROM /home/tnv/ai_infratest/models/Qwen2.5-Coder-7B-Instruct-Q2_K.gguf
SYSTEM "You are an expert embedded Linux developer specializing in C programming, Linux kernel drivers, RTOS, HiSilicon hi3518 and Rockchip SoCs. Always write clean efficient C code with error handling and comments."
PARAMETER num_gpu 33
PARAMETER num_ctx 2048
EOF

cat > ~/ai_infratest/modelfiles/deepseek-r1 << 'EOF'
FROM /home/tnv/ai_infratest/models/deepseek-r1-distill-qwen-1.5b-q4_0.gguf
SYSTEM "You are an expert embedded Linux developer. Think step by step before answering."
PARAMETER num_gpu 99
PARAMETER num_ctx 2048
EOF
```
**ollama create:**

```
ollama create qwen-coder-3b -f ~/ai_infratest/modelfiles/qwen-coder-3b
ollama create qwen-coder-7b -f ~/ai_infratest/modelfiles/qwen-coder-7b
ollama create deepseek-r1-1.5b -f ~/ai_infratest/modelfiles/deepseek-r1
```
**Проверка:**
```
ollama list
```

### Шаг 4 — быстрый тест что модель работает на GPU:
**Терминал №1:**
```
ollama run qwen-coder-3b "напиши hello world на C"
```
**Терминал №2:**
```
// Проверить что LLM работает на GPU
nvidia-smi
```

## open-webui

### Шаг 5 — создать docker-compose.yml и запустить open-webui:
```
mkdir -p ~/ai_infratest/docker

cat > ~/ai_infratest/docker/docker-compose.yml << 'EOF'
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://host.docker.internal:11434
    volumes:
      - open-webui:/app/backend/data
    extra_hosts:
      - "host.docker.internal:host-gateway"
    restart: always
volumes:
  open-webui:
EOF

cd ~/ai_infratest/docker
docker compose up -d
```
**Проверка:**
```
docker ps
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000
```
