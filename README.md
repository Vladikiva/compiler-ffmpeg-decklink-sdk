# Компиляция FFmpeg с поддержкой карт DeckLink на Ubuntu/Debian

**Пошаговая инструкция** по установке драйвера *Blackmagic* и сборке *FFmpeg*.

## Шаг 1. Установка драйвера и подготовка SDK

- 1. Скачайте с официального сайта Blackmagic Design *(раздел Support -> Capture & Playback)* два архива для Linux:
     ➕ **Desktop Video** (драйверы и утилиты управления).
     ➕ **Desktop Video SDK** (заголовочные файлы для компиляции).
- 2. Установить *.deb* пакеты драйвера из распакованного архива Desktop Video (обычно из папки deb/x86_64/):

```bash
sudo dpkg -i desktopvideo_*.deb
sudo apt-get install -f -y
sudo reboot
```
- 3. После перезагрузки проверьте, что система видит плату:
```bash
BlackmagicFirwareUpdater status
```
- 4. Скопируйте заголовочные файлы SDK в рабочую директорию без пробелов в названии (это критично, так как стандартное имя папки архива с пробелами ломает сборку):
```bash
mkdir -p ~/BMD_SDK/Linux/include
cp -r ~/"Blackmagic DeckLink SDK"*/Linux/include/* ~/BMD_SDK/Linux/include/
```
(Убедиться, что файл DeckLinkAPI.h лежит прямо по пути ~/BMD_SDK/Linux/include/DeckLinkAPI.h).

## Шаг 2. Установка системных зависимостей
   Установите компиляторы и все необходимые библиотеки для работы видео- и аудиокодеков, а также протоколов вещания (SRT,OpenSSL):
```bash
sudo apt-get update
sudo apt-get install -y \
  autoconf automake build-essential cmake git-core libass-dev \
  libfreetype6-dev libssl-dev libmp3lame-dev libsdl2-dev \
  libtool libva-dev libvdpau-dev libvorbis-dev libxcb1-dev \
  libxcb-shm0-dev libxcb-xfixes0-dev meson ninja-build pkg-config \
  texinfo wget yasm nasm zlib1g-dev libx264-dev libx265-dev \
  libnuma-dev libvpx-dev libfdk-aac-dev libopus-dev libsrt-openssl-dev
```
## Шаг 3. Скачивание и сборка FFmpeg с поддержкой DeckLink
- 1. Создайте рабочие директории и склонируйте свежие исходники FFmpeg:
```bash
mkdir -p ~/ffmpeg_sources ~/bin
cd ~/ffmpeg_sources
git clone --depth 1 https://ffmpeg.org ffmpeg
cd ffmpeg
```
- 2. Сконфигурируйте сборку одной командой (в ней указаны пути к SDK, стандарт - std=c++14 и библиотека -ldl, необходимые для современных версий Blackmagic SDK):
```bash
./configure --prefix="$HOME/ffmpeg_build" --extra-cflags="-I$HOME/ffmpeg_build/include -I$HOME/BMD_SDK/Linux/include" --extra-cxxflags="-I$HOME/BMD_SDK/Linux/include -std=c++14" --extra-ldflags="-L$HOME/ffmpeg_build/lib" --extra-libs="-lpthread -lm -ldl" --ld="g++" --bindir="$HOME/bin" --enable-gpl --enable-openssl --enable-libass --enable-libfdk-aac --enable-libfreetype --enable-libmp3lame --enable-libopus --enable-libsrt --enable-libvorbis --enable-libvpx --enable-libx264 --enable-libx265 --enable-nonfree --enable-decklink
```
- 3. Запустите компиляцию и установите готовые бинарники в систему:
```bash
make -j$(nproc)
make install
sudo cp "$HOME/bin/ffmpeg" "$HOME/bin/ffprobe" /usr/local/bin/
```
## Шаг 4. Проверка работы и базовые команды
   ➕ Список подключенных устройств Blackmagic:
```bash
ffmpeg -sources decklink
```
➕ Список поддерживаемых видеоформатов и кодов разрешений карты:
```bash
ffmpeg -f decklink -list_formats 1 -i " Название карты"
```
➕ Захват сигнала в файл MP4 (пример для 1080p50 - Hp50):
```bash
ffmpeg -format_code Hp50 -f decklink -i "Название карты" -c:v libx264 -preset veryfast -crf 20 -pix_fmt yuv420p -c:a aac -b:a 192k output.mp4
```
➕ Прямая трансляция потока по протоколу SRT:
```bash
ffmpeg -format_code Hp50 -f decklink -i "Название карты" -c:v libx264 -preset ultrafast -tune zerolatency -b:v 8000k -pix_fmt yuv420p -c:a aac -b:a 192k -f mpegts "srt://IP_СЕРВЕРА:ПОРТ?mode=caller"
```
➕ Вывод видеофайла на выход карты:
```bash
ffmpeg -re -i input.mp4 -f decklink -pix_fmt uyvy422 "Название карты"
```
(Совет: не удаляйте папку ~/ffmpeg_sources/ffmpeg - при обновлении драйвера Blackmagic достаточно обновить файлы в ~/BMD_SDK/Linux/include и заново выполнить make clean && ./configure ... ** make -j$(nproc) ).
