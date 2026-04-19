---

🧱 Fedora KDE как функциональный хост + изолированная dev‑среда
Podman (rootless) + Distrobox + Arch dev‑container + VS Code (Microsoft build)

---

📌 Цель

Построить систему с чётким разделением ролей:

- 🖥 Host (Fedora KDE) — минимальный, стабильный, безопасный  
- 📦 Контейнеры (Podman + Distrobox) — разработка, тесты  
- 🧪 VM (KVM/libvirt) — максимальная изоляция  
- 💾 Btrfs subvolumes — контроль состояния и быстрые откаты  

---

🧠 Архитектура

`
BTRFS
│
├── /var/lib/libvirt                    ← VM (KVM)
├── /var/lib/containers                 ← rootful (не используется)
│
└── /home/engineer
    ├── dev-projects                    ← проекты
    └── .local/share
        ├── containers                  ← rootless podman
        └── distrobox/dev-home          ← HOME контейнера
`

---

💾 Btrfs subvolumes

Создание:

`bash
sudo btrfs subvolume create /home/engineer/.local/share/containers
sudo btrfs subvolume create /home/engineer/.local/share/distrobox/dev-home
sudo btrfs subvolume create /home/engineer/dev-projects
`

Проверка:

`bash
sudo btrfs subvolume list /
`

---

📦 Podman (rootless)

Проверка storage:

`bash
podman info --format '{{.Store.GraphRoot}}'
`

Ожидаемый результат:

`
/home/engineer/.local/share/containers/storage
`

✔ Используется rootless Podman  
✔ /var/lib/containers не используется  

---

📦 Создание dev‑контейнера Distrobox

Удаление старого:

`bash
distrobox rm -f dev
`

Создание нового:

`bash
distrobox create \
  --name dev \
  --image docker.io/library/archlinux:latest \
  --home /home/engineer/.local/share/distrobox/dev-home
`

Список контейнеров:

`bash
distrobox list
`

---

🚀 Вход в контейнер

`bash
distrobox enter dev
`

---

⚙️ Базовая настройка контейнера

`bash
sudo pacman -Syu --noconfirm
sudo pacman -S --needed --noconfirm \
  base-devel git curl wget unzip tar xdg-utils \
  python python-pip python-virtualenv \
  python-debugpy python-pytest \
  ruff black pyright
`

---

⚡ Установка yay (AUR helper)

`bash
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
`


Оптимизация yay и сборки:

Файл сборки:
/etc/makepkg.conf

1. Оптимизация под твое железо (-march)

У тебя: -march=x86-64 -mtune=generic. Это значит: «собирай код, который запустится на любом 64-битном процессоре за последние 15 лет».
Как лучше: Замени на -march=native.
Компилятор определит твой конкретный процессор (например, Ryzen 5 или Core i7) и задействует все его инструкции (AVX, AVX2 и т.д.). Программы будут работать быстрее.

2. Уровень оптимизации (-O2 vs -O3)
У тебя: -O2. Это «золотая середина».
Как лучше: Для некоторых вычислительных задач можно поставить -O3, но это увеличивает размер бинарника и иногда ломает сборку. Для 99% случаев -O2 (через букву O, не ноль!) — оптимально.
В твоем тексте опечатка: написано -02 (ноль), убедись, что в конфиге стоит именно -O2 (буква O).

3. Удаление лишнего для домашнего использования
Флаги типа -fstack-clash-protection, -fcf-protection и _FORTIFY_SOURCE=3 — это крутая защита от хакерских атак (переполнение стека и т.д.).
Если ты собираешь тяжелый софт (например, движок или браузер) и тебе важна каждая секунда времени сборки, их можно убрать, но для обычной работы в Distrobox их лучше оставить — безопасность лишней не бывает.

4. Frame Pointers (для отладки)
У тебя: -fno-omit-frame-pointer. Это позволяет профилировщикам и отладчикам (вроде того же debugpy или perf) точнее показывать дерево вызовов.
Вердикт: Для разработчика это полезно, оставляй.

Файл yay
'bash
nano ~/.config/yay/config.json
'

Пример:

`json
{
  "aururl": "https://aur.archlinux.org",
  "buildDir": "/tmp",
  "editor": "nano",
  "makepkgbin": "makepkg",
  "pacmanbin": "sudo pacman",
  "sortby": "votes",
  "sudobin": "sudo"
}
`

---

🧩 Установка VS Code (официальный Microsoft build)

`bash
yay -S visual-studio-code-bin
`

Проверка:

`bash
code --version
`

---

🖥 Экспорт VS Code в KDE

`bash
distrobox-export --app code
`

✔ VS Code появляется в системе как обычное приложение  
✔ но работает внутри контейнера  

---

🐍 Python‑проект

Создание проекта:

`bash
mkdir -p ~/dev-projects/python-test
cd ~/dev-projects/python-test
`

Виртуальное окружение:

`bash
python -m venv .venv
source .venv/bin/activate
`

Обновление pip:

`bash
python -m pip install --upgrade pip
`

Установка зависимостей:

`bash
pip install debugpy pytest ruff black
`

---

⚙️ Настройка VS Code

.vscode/settings.json:

`json
{
  "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
  "python.terminal.activateEnvironment": true,
  "python.testing.pytestEnabled": true,
  "editor.formatOnSave": true
}
`

---

🐞 Debug конфигурация

.vscode/launch.json:

`json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Python: текущий файл",
      "type": "debugpy",
      "request": "launch",
      "program": "${file}",
      "console": "integratedTerminal"
    }
  ]
}
`

---

💾 Снапшоты Btrfs

Создание snapshot:

`bash
sudo btrfs subvolume snapshot \
  /home/engineer/.local/share/distrobox/dev-home \
  /home/engineer/dev-home-backup
`

Откат:

`bash
rm -rf ~/.local/share/distrobox/dev-home
mv ~/dev-home-backup ~/.local/share/distrobox/dev-home
`

---

🔐 Модель безопасности

| Уровень | Инструмент |
|--------|------------|
| Host | Fedora + SELinux |
| Контейнеры | Podman (rootless) |
| Изоляция среды | Distrobox |
| Максимальная изоляция | KVM / QEMU |

---

📊 Итог

✔ Чистый host  
✔ Изолированная dev‑среда  
✔ Контроль через Btrfs  
✔ VS Code не трогает систему  
✔ Контейнеры вместо «засорения» ОС  

---

🚀 Дальнейшие шаги

- [ ] Разделение контейнеров: dev, dirty, clean
- [ ] Отдельный контейнер для браузинга
- [ ] Автоматизация снапшотов
- [ ] Подключение VPN/Tor на уровне контейнеров

---

⚠️ Важно

НЕ смешивать rootless и rootful Podman.

---

🧠 Концепция

- Host = стабильность  
- Containers = гибкость  
- VM = изоляция  
- Btrfs = контроль  

---
