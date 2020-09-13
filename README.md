# OpenWRT-Wireguard
### Это слияние двух интрукций
1. [На хабре](https://habr.com/ru/post/440030/)
2. [На overclockers](https://overclockers.ru/blog/Indigo81/show/30877/wireguard-openwrt-unbound-divnyj-novyj-mir-vpn)
3. И мои + чужие доработки людей в комментариях на хабре
### Описание конфигураций
1. Сервер Debian 10 в нидерландах, чистый
2. Роутер Zyxel Keenetic Omni, прошивка OpenWRT 19.07.4

### Настройка сервера
Добавляем репозиторий Wireguard

```sh
$ echo "deb http://deb.debian.org/debian/ unstable main" >/etc/apt/sources.list.d/unstable.list
$ printf 'Package: *\nPin: release a=unstable\nPin-Priority: 90\n' >/etc/apt/preferences.d/limit-unstable
$ apt update
$ apt full-upgrade -y
```

Устанвливаем часовый пояс

```sh
$ apt install tzdata
$ dpkg-reconfigure tzdata
```
Установка заголовков ядра
```sh
$ apt install "linux-headers-$(uname -r)" -y
```
Установка wireguard, qrencode, curl, unbound, ufw
```sh
$ apt install wireguard qrencode curl unbound unbound-host ufw -y
```


Файл концигурации wireguard, `/etc/wireguard/wg0.conf`
```sh
[Interface]
Address = 10.0.0.1/8
Address = fd42:42:42::1/48
SaveConfig = false
PrivateKey = server_private_key

PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; ip6tables -A FORWARD -i %i -j ACCEPT; ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; ip6tables -D FORWARD -i %i -j ACCEPT; ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
ListenPort = 51820

[Peer] # Роутер
PublicKey = router_client_public_key
AllowedIPs = 10.0.0.0/16, fd42:42:42::/60

[Peer] # Телефон или другое устройство
PublicKey = other_client_public_key
AllowedIPs = 10.50.0.1/32, fd42:42:42:ffff::1/128
```

Заменить `eth0` на свой, проверить коммандой можно: `ip a`

Создаем ключи, будут лежать в `/root`:
- Для сервера: `wg genkey | tee server_private_key | wg pubkey > server_public_key`
- Для роутера: `wg genkey | tee router_client_private_key | wg pubkey > router_client_public_key`
- Для другого устройства: `wg genkey | tee other_client_private_key | wg pubkey > other_client_public_key`

### Настройка DNS
Заходим в папку `/etc/unbound/unbound.conf.d/`, удаляем содержащиеся здесь файлы
