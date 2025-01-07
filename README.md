# openvpn-config
репозиторий_настройки_openvpn-config_для_личного_пользования
Да, вы можете использовать GitHub для хранения конфигураций и скриптов OpenVPN, чтобы в будущем легко развернуть их на любом устройстве. Это удобно для управления версиями, совместной работы и резервного копирования. Давайте начнем с самого начала и настроим OpenVPN на вашем сервере Oracle Linux 8 с использованием вашего аккаунта GitHub.

---

### **1. Подготовка GitHub репозитория**
1. **Создайте новый репозиторий на GitHub:**
   - Перейдите на [GitHub](https://github.com).
   - Нажмите на кнопку **New** (Создать новый репозиторий).
   - Назовите репозиторий, например, `openvpn-config`.
   - Выберите **Public** (если хотите, чтобы он был открытым) или **Private** (если хотите, чтобы он был приватным).
   - Нажмите **Create repository**.

2. **Клонируйте репозиторий на ваш сервер:**
   - На вашем сервере Oracle Linux установите Git:
     ```bash
     sudo dnf install git -y
     ```
   - Склонируйте репозиторий:
     ```bash
     git clone https://github.com/ВАШ_АККАУНТ/ВАШ_РЕПОЗИТОРИЙ.git
     ```
     Например:
     ```bash
     git clone https://github.com/username/openvpn-config.git
     ```
   - Перейдите в директорию репозитория:
     ```bash
     cd openvpn-config
     ```

---

### **2. Установка OpenVPN и Easy-RSA**
1. Установите OpenVPN и Easy-RSA:
   ```bash
   sudo dnf install openvpn easy-rsa -y
   ```

2. Создайте директорию для Easy-RSA и скопируйте туда шаблоны:
   ```bash
   mkdir ~/easy-rsa
   ln -s /usr/share/easy-rsa/3/* ~/easy-rsa/
   cd ~/easy-rsa
   ```

3. Инициализируйте PKI (Public Key Infrastructure):
   ```bash
   ./easyrsa init-pki
   ```

4. Создайте корневой сертификат (CA):
   ```bash
   ./easyrsa build-ca
   ```

5. Создайте сертификат и ключ для сервера:
   ```bash
   ./easyrsa gen-req server nopass
   ./easyrsa sign-req server server
   ```

6. Создайте Diffie-Hellman параметры:
   ```bash
   ./easyrsa gen-dh
   ```

7. Создайте ключ для TLS:
   ```bash
   openvpn --genkey --secret ta.key
   ```

---

### **3. Настройка OpenVPN сервера**
1. Скопируйте созданные файлы в директорию `/etc/openvpn/server/`:
   ```bash
   sudo cp ~/easy-rsa/pki/ca.crt /etc/openvpn/server/
   sudo cp ~/easy-rsa/pki/issued/server.crt /etc/openvpn/server/
   sudo cp ~/easy-rsa/pki/private/server.key /etc/openvpn/server/
   sudo cp ~/easy-rsa/pki/dh.pem /etc/openvpn/server/
   sudo cp ~/easy-rsa/ta.key /etc/openvpn/server/
   ```

2. Создайте конфигурационный файл для OpenVPN сервера:
   ```bash
   sudo nano /etc/openvpn/server/server.conf
   ```

   Пример конфигурации:
   ```ini
   port 1194
   proto udp
   dev tun
   ca ca.crt
   cert server.crt
   key server.key
   dh dh.pem
   server 10.8.0.0 255.255.255.0
   ifconfig-pool-persist /var/log/openvpn/ipp.txt
   push "redirect-gateway def1 bypass-dhcp"
   push "dhcp-option DNS 8.8.8.8"
   push "dhcp-option DNS 8.8.4.4"
   keepalive 10 120
   tls-auth ta.key 0
   cipher AES-256-CBC
   user nobody
   group nobody
   persist-key
   persist-tun
   status /var/log/openvpn/openvpn-status.log
   verb 3
   explicit-exit-notify 1
   ```

3. Разрешите пересылку пакетов через сервер:
   ```bash
   sudo nano /etc/sysctl.conf
   ```
   Добавьте или раскомментируйте строку:
   ```ini
   net.ipv4.ip_forward=1
   ```
   Примените изменения:
   ```bash
   sudo sysctl -p
   ```

4. Настройте iptables для маршрутизации трафика:
   ```bash
   sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
   sudo iptables-save | sudo tee /etc/sysconfig/iptables
   ```

5. Запустите OpenVPN и добавьте его в автозагрузку:
   ```bash
   sudo systemctl start openvpn-server@server
   sudo systemctl enable openvpn-server@server
   ```

---

### **4. Настройка клиента**
1. Создайте сертификат и ключ для клиента:
   ```bash
   cd ~/easy-rsa
   ./easyrsa gen-req client1 nopass
   ./easyrsa sign-req client client1
   ```

2. Скопируйте необходимые файлы в директорию вашего репозитория:
   ```bash
   cp ~/easy-rsa/pki/ca.crt ~/openvpn-config/
   cp ~/easy-rsa/pki/issued/client1.crt ~/openvpn-config/
   cp ~/easy-rsa/pki/private/client1.key ~/openvpn-config/
   cp ~/easy-rsa/ta.key ~/openvpn-config/
   ```

3. Создайте конфигурационный файл для клиента:
   ```bash
   nano ~/openvpn-config/client.ovpn
   ```

   Пример конфигурации:
   ```ini
   client
   dev tun
   proto udp
   remote YOUR_SERVER_IP 1194
   resolv-retry infinite
   nobind
   persist-key
   persist-tun
   ca ca.crt
   cert client1.crt
   key client1.key
   tls-auth ta.key 1
   cipher AES-256-CBC
   verb 3
   ```

   Замените `YOUR_SERVER_IP` на IP-адрес вашего сервера.

4. Добавьте файлы в репозиторий и загрузите их на GitHub:
   ```bash
   git add .
   git commit -m "Initial OpenVPN configuration"
   git push origin main
   ```

---

### **5. Использование конфигурации на другом устройстве**
1. Клонируйте репозиторий на устройство, где нужно настроить OpenVPN:
   ```bash
   git clone https://github.com/ВАШ_АККАУНТ/ВАШ_РЕПОЗИТОРИЙ.git
   ```

2. Используйте файл `client.ovpn` для подключения к серверу через OpenVPN.

---

Теперь у вас есть полностью настроенный OpenVPN сервер, а все конфигурации и ключи сохранены в вашем GitHub репозитории. Вы можете легко развернуть их на любом устройстве в будущем.

приступаем к реализации данного проекта
