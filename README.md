## Описание файлов в директории
logFileFull.log - полный лог выполнения  

Vagrant_folder - все что понадобится для поднятия ВМ и краткое описание файлов в ней  
Vagrantfile - вагрант файл  
provisioning - директория с файлами провижинга ВМ backupServer и client
provisioning/
```
├── playbook.yml
└── roles
    ├── backupServer
    │   ├── defaults
    │   │   └── main.yml
    │   ├── files
    │   ├── handlers
    │   │   └── main.yml
    │   ├── meta
    │   │   └── main.yml
    │   ├── README.md
    │   ├── tasks
    │   │   └── main.yml
    │   ├── templates
    │   ├── tests
    │   │   ├── inventory
    │   │   └── test.yml
    │   └── vars
    │       └── main.yml
    └── client
        ├── defaults
        │   └── main.yml
        ├── files
        │   ├── borg-backup.service
        │   └── borg-backup.timer
        ├── handlers
        │   └── main.yml
        ├── meta
        │   └── main.yml
        ├── README.md
        ├── tasks
        │   └── main.yml
        ├── templates
        ├── tests
        │   ├── inventory
        │   └── test.yml
        └── vars
            └── main.yml
```

## Описание как запустить виртуальную машину (кратко)
```
vagrant up
vagrant ssh client
```

## VagrantFile только важное
К ВМ backupServer подключаем диск для хранение бекапов через провайдера virtual box
```
   backupServer.vm.provider :virtualbox do |vb|
      needsController = false
      unless File.exist?('./sata1.vdi')
        vb.customize ['createhd', '--filename', './sata1.vdi', '--variant', 'Fixed', '--size', 2048]
        needsController =  true
      end
      vb.customize ['storageattach', :id, '--storagectl', 'IDE', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', './sata1.vdi']
    end
```
На ВМ inetRouter2 настраиваем проброс порта. Любой входящий трафик на порт 8080, в PREROUTING меняем адрес назаначения 192.168.0.2:80. А в POSTROUTING меняем адрес отправителя.
```
        when "inetRouter2"
          box.vm.provision "shell", run: "always", inline: <<-SHELL
            echo 1 > /proc/sys/net/ipv4/ip_forward
            iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.0.2:80
            iptables -t nat -I POSTROUTING -p tcp --dst 192.168.0.2 --dport 80 -j SNAT --to 192.168.254.1
            ip r add 192.168.0.2/32 via 192.168.254.2
            SHELL
```

## ansible playbook только важное
### client
Генерируем ключ для root
```
- name: create ssh key to root
  user:
    name: root
    generate_ssh_key: yes
```
Копируем публичный ключ на головную машину.
```
- name: Copy sert
  fetch:
    src: /root/.ssh/id_rsa.pub
    dest: ../
    flat: yes
```
Потом создаем файлы, для systemd, демона и таймера. Включаем таймер. Изменяем опцию StrictHostKeyChecking демона sshd, чтобы небыло запроса подтвержения подключения к backupServer.

### backupServer
Копируем с головной машины публичный ключ, для доступа с ВМ client. Готовим ранее созданный диск для бекапов. И создаем символьную ссылку /var/backup на наш диск для бекапов. Проводим инициализацию репозитория и шифруем его паролем 'Otus1234'.

## Логи
### Лог бекапа
systemd неплохо логирует выполнение демона
```
Jul 17 11:35:01 localhost systemd: Starting Borg Backup...
Jul 17 11:35:03 localhost borg: ------------------------------------------------------------------------------
Jul 17 11:35:03 localhost borg: Archive name: etc-2022-07-17_11:35:01
Jul 17 11:35:03 localhost borg: Archive fingerprint: 4c87d4204fbff81790508e10bb100fb982504a3a8f5cf098159958394d1077d0
Jul 17 11:35:03 localhost borg: Time (start): Sun, 2022-07-17 11:35:02
Jul 17 11:35:03 localhost borg: Time (end):   Sun, 2022-07-17 11:35:03
Jul 17 11:35:03 localhost borg: Duration: 0.48 seconds
Jul 17 11:35:03 localhost borg: Number of files: 1700
Jul 17 11:35:03 localhost borg: Utilization of max. archive size: 0%
Jul 17 11:35:03 localhost borg: ------------------------------------------------------------------------------
Jul 17 11:35:03 localhost borg: Original size      Compressed size    Deduplicated size
Jul 17 11:35:03 localhost borg: This archive:               28.43 MB             13.49 MB                619 B
Jul 17 11:35:03 localhost borg: All archives:               56.85 MB             26.99 MB             11.84 MB
Jul 17 11:35:03 localhost borg: Unique chunks         Total chunks
Jul 17 11:35:03 localhost borg: Chunk index:                    1282                 3394
Jul 17 11:35:03 localhost borg: ------------------------------------------------------------------------------
Jul 17 11:35:06 localhost systemd: Started Borg Backup.
Jul 17 11:38:04 localhost systemd: Stopped Borg Backup.
```
### Лог восстановления
Останавливаем таймер.
```
[root@client ~]# systemctl stop borg-backup.timer
```
Узнаем имя бекапа и монтируем его.
```
[root@client ~]# borg list borg@192.168.11.160:/var/backup
Enter passphrase for key ssh://borg@192.168.11.160/var/backup:
etc-2022-07-17_11:35:01              Sun, 2022-07-17 11:35:02 [4c87d4204fbff81790508e10bb100fb982504a3a8f5cf098159958394d1077d0]
[root@client ~]# borg mount borg@192.168.11.160:/var/backup::etc-2022-07-17_11:35:01 /mnt
```
Перемещаем файлы директории /etc в ранее созданную директорию. Копируем файлы из смонтированного бекапа обратно в /etc.
```
[root@client ~]# mkdir /tmp/etc/
[root@client ~]# mv /etc/* /tmp/etc
/bin/cp -prf /mnt/etc/* /etc/
```

📚Домашнее задание/проектная работа разработано(-на) для курса ["Administrator Linux. Professional"](https://otus.ru/lessons/linux-professional/)