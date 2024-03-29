<h1 align="center"><i>Экзаменационное задание по ПМ.02. Организация сетевого администрирования</i></h1>
<i><strong>Сценарий.</strong></i><br>
<i>Вы являетесь системным администратором компании в которой имеется гетерогенная сеть с использованием решений различных производителей. Серверная на основе Windows Server 2019 (Desktop и Core) AltLinux Server, есть клиентские машины на основе Windows 10 Pro и AltLinux.  Часть сервисов уже развёрнута на компьютерах организации. В головном офисе компании имеется роутер для подключения к сети провайдера. Провайдер предоставляет подключение к глобальной сети, шлюз по умолчанию, а также выделенные адреса для подключения. В компании имеются сотрудники, которые работают на удалёнке на корпоративных ПК на основе Simply Linux, которые должны иметь доступ к информационной системе компании. Кроме того, в планах руководства есть желание построить корпоративный портал на отказоустойчивой инфраструктуре.</i>
<i>Конечной целью является полноценное функционирование инфраструктуры предприятия в пределах соответствующих регионов.
Имеющаяся инфраструктура представлена на диаграмме:</i>
<br>
<br>

![Image alt](https://github.com/NewErr0r/Qualification_Exam/blob/main/topologya.png)

<strong> Таблица адресации: </strong>
<br>

![Image alt](https://github.com/NewErr0r/Qualification_Exam/blob/main/tableaddressing.png)

<i>Ваша задача донастроить инфраструктуру организации в соответствии с требованиями руководства компании.</i>
<br>
<i>Сервер DC является контроллером домена на нём развёрнуты сервисы Active Directory(домен – Oaklet.org), DNS.</i>
<br>


<h1>Базовая конфигурация (подготовительные настройки):</h1>
<ul>
    <li><strong>FW (name, nameserver, gateway, addressing, nat, dhcp-relay)</strong></li>
</ul>
<br>
<pre>
set system host-name FW

set interface ethernet eth1 address 172.20.0.1/24
set interface ethernet eth2 address 172.20.2.1/23

set nat source rule 1 outboun-interface eth0
set nat source rule 2 outboun-interface eth0
set nat source rule 1 source address 172.20.0.0/24
set nat source rule 2 source address 172.20.2.0/23
set nat source rule 1 translation address masquerade
set nat source rule 2 translation address masquerade

set service dhcp-relay interface eth1
set service dhcp-relay interface eth2
set service dhcp-relay server 172.20.0.100
set service dhcp-relay relay-options relay-agents-packets discard
</pre>

<ul>
    <li><strong>DC (DNS)</strong></li>
</ul>
<br>
<pre>
Add-DnsServerPrimaryZone -NetworkId "172.20.0.0/24" -ReplicationScope Domain
Add-DnsServerPrimaryZone -NetworkId "172.20.2.0/24" -ReplicationScope Domain
Add-DnsServerPrimaryZone -NetworkId "172.20.3.0/24" -ReplicationScope Domain
Add-DnsServerResourceRecordPtr -ZoneName 0.20.172.in-addr.arpa -Name 100 -PtrDomainName dc.Oaklet.org
Add-DnsServerResourceRecordA -Name "FS" -ZoneName "Oaklet.org" -AllowUpdateAny -IPv4Address "172.20.0.200" -CreatePtr
Add-DnsServerResourceRecordA -Name "SRV" -ZoneName "Oaklet.org" -AllowUpdateAny -IPv4Address "172.20.3.100" -CreatePtr<br>
Add-DnsServerResourceRecordCName -Name "www" -HostNameAlias "SRV.Oaklet.org" -ZoneName "Oaklet.org"
Add-DnsServerPrimaryZone -Name first -ReplicationScope "Forest" –PassThru
Add-DnsServerResourceRecordA -Name "app" -ZoneName "first" -AllowUpdateAny -IPv4Address "200.100.100.200"
</pre>

<ul>
    <li><strong>FS (Disabled Firewall)</strong></li>
</ul>
<br>
<pre>
powershell
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled false
</pre>
<pre>
Add-Computer
    Администратор
    P@ssw0rd
        Oaklet.org
Restart-Computer
</pre>

<ul>
    <li><strong>SRV (name, addressing)</strong></li>
</ul>
<br>
<pre>
su -
hostnamectl set-hostname SRV.Oaklet.org
reboot
</pre>
<pre>
ЦУС -> Сеть -> Ethernet-интерфейсы
IP: 172.20.3.100/23
Шлюз по умолчанию: 172.20.2.1
DNS-серверы: 172.20.0.100 77.88.8.8
Домены поиска: Oaklet.org
</pre>
<pre>
su -
apt-get update
apt-get install -y task-auth-ad-sssd
system-auth write ad Oaklet.org SRV Oaklet 'Администратор' 'P@ssw0rd'
reboot
</pre>

<ul>
    <li>APP-V (name, addressing, nat)</strong></li>
</ul>
<br>
<pre>
hostnamectl set-hostname APP-V

mkdir /etc/net/ifaces/enp0s8
cp /etc/net/ifaces/enp0s3/options /etc/net/ifaces/enp0s8

echo 10.116.0.10/14 >> /etc/net/ifaces/enp0s8/ipv4address
systemctl restart network
ip link set up enp0s3
ip link set up enp0s8

echo nameserver 77.88.8.8 > /etc/resolv.conf
apt-get update
apt-get install firewalld -y
systemctl enable --now firewalld

firewall-cmd --permanent --zone=trusted --add-interface=enp0s8
firewall-cmd --permanent --add-masquerade
firewall-cmd --reload

echo net.ipv4.ip_forward=1 >> /etc/sysctl.conf
sysctl -p
</pre>

<ul>
    <li>APP-L (name, addressing)</strong></li>
</ul>
<br>
<pre>
hostnamectl set-hostname APP-L
echo 10.116.0.20/14 >> /etc/net/ifaces/enp0s3/ipv4address
echo default via 10.116.0.10 > /etc/net/ifaces/enp0s3/ipv4route
systemctl restart network
ip link set up enp0s3
echo nameserver 77.88.8.8 > /etc/resolv.conf
</pre>

<ul>
    <li>APP-R (name, addressing)</strong></li>
</ul>
<br>
<pre>
hostnamectl set-hostname APP-R
echo 10.116.0.30/14 >> /etc/net/ifaces/enp0s3/ipv4address
echo default via 10.116.0.10 > /etc/net/ifaces/enp0s3/ipv4route
systemctl restart network
ip link set up enp0s3
echo nameserver 77.88.8.8 > /etc/resolv.conf
</pre>

<ul>
    <li>CLI-R (name, addressing)</strong></li>
</ul>
<br>
<pre>
su -
hostnamectl set-hostname CLI-R
reboot

ЦУС -> Сеть -> Ethernet-интерфейсы
IP: 200.100.100.10/24
Шлюз по умолчанию: 200.100.100.254
DNS-серверы: 77.88.8.8 172.20.0.100 

su - 
ip link set up enp0s3
</pre>

<h1>Элементы доменной инфраструктуры:</h1>
<ul>
    <li><strong>На сервере контроллера домена необходимо развернуть следующую организационную структуру:</strong></li>
</ul>
<br>

![Image alt](https://github.com/NewErr0r/Qualification_Exam/blob/main/departament.png)
<br>

<pre>
New-ADOrganizationalUnit -Name ADM
New-ADOrganizationalUnit -Name Sales
New-ADOrganizationalUnit -Name Delivery
New-ADOrganizationalUnit -Name Development

New-ADGroup "ADM" -path 'OU=ADM,DC=Oaklet,DC=org' -GroupScope Global -PassThru –Verbose
New-ADGroup "Sales" -path 'OU=Sales,DC=Oaklet,DC=org' -GroupScope Global -PassThru –Verbose
New-ADGroup "Delivery" -path 'OU=Delivery,DC=Oaklet,DC=org' -GroupScope Global -PassThru –Verbose
New-ADGroup "Frontend" -path 'OU=Development,DC=Oaklet,DC=org' -GroupScope Global -PassThru –Verbose
New-ADGroup "Backend" -path 'OU=Development,DC=Oaklet,DC=org' -GroupScope Global -PassThru –Verbose

New-ADUser -Name "Director" -UserPrincipalName "Director@Oaklet.org" -Path "OU=ADM,DC=Oaklet,DC=org" -AccountPassword(ConvertTo-SecureString P@ssw0rd -AsPlainText -Force) -Enabled $true
New-ADUser -Name "Secretary" -UserPrincipalName "Secretary@Oaklet.org" -Path "OU=ADM,DC=Oaklet,DC=org" -AccountPassword(ConvertTo-SecureString P@ssw0rd -AsPlainText -Force) -Enabled $true
New-ADUser -Name "Alice" -UserPrincipalName "Alice@Oaklet.org" -Path "OU=Sales,DC=Oaklet,DC=org" -AccountPassword(ConvertTo-SecureString P@ssw0rd -AsPlainText -Force) -Enabled $true
New-ADUser -Name "Bob" -UserPrincipalName "Bob@Oaklet.org" -Path "OU=Sales,DC=Oaklet,DC=org" -AccountPassword(ConvertTo-SecureString P@ssw0rd -AsPlainText -Force) -Enabled $true
New-ADUser -Name "Polevikova" -UserPrincipalName "Polevikova@Oaklet.org" -Path "OU=Delivery,DC=Oaklet,DC=org" -AccountPassword(ConvertTo-SecureString P@ssw0rd -AsPlainText -Force) -Enabled $true
New-ADUser -Name "Morgushko" -UserPrincipalName "Morgushko@Oaklet.org" -Path "OU=Development,DC=Oaklet,DC=org" -AccountPassword(ConvertTo-SecureString P@ssw0rd -AsPlainText -Force) -Enabled $true
New-ADUser -Name "Radjkovith" -UserPrincipalName "Radjkovith@Oaklet.org" -Path "OU=Development,DC=Oaklet,DC=org" -AccountPassword(ConvertTo-SecureString P@ssw0rd -AsPlainText -Force) -Enabled $true

New-ADUser -Name "smb" -UserPrincipalName "smb@Oaklet.org" -Path "DC=Oaklet,DC=org" -AccountPassword(ConvertTo-SecureString P@ssw0rd -AsPlainText -Force) -Enabled $true

Add-AdGroupMember -Identity ADM Director, Secretary
Add-AdGroupMember -Identity Sales Alice, Bob
Add-AdGroupMember -Identity Delivery Polevikova
Add-AdGroupMember -Identity Frontend Morgushko
Add-AdGroupMember -Identity Backend Radjkovith
</pre>
<br>

<ul>
    <li><strong>Должны быть настроены следующие GPO:</strong></li>
    <ul>
        <li>отключить OneDrive ( имя политики onedrive);</li>
        <li>Запретить чтение информации со съёмных носителей ( имя политики removable media);</li>
        <li>Отключить использование камер (имя политики camera);</li>
        <li>Запретить любые изменения персонализации рабочего стола ( имя политики desktop);</li>
    </ul>
</ul>
<pre>
New-GPO -Name "onedrive" | New-GPLink -Target "DC=Oaklet,DC=org"
Конфигурация компьютера -> Политики -> Административные шаблоны -> Компоненты Windows -> OneDrive -> Запретить использование OneDrive для хранения файлов (включить)
<br>
New-GPO -Name "removable media" | New-GPLink -Target "DC=Oaklet,DC=org"
Конфигурация компьютера -> Политики -> Административные шаблоны -> Система -> Доступ к съемным запоминающим устройствам -> Съемные запоминающие устройства всех классов: Запретить любой доступ (включить)
Конфигурация пользователя -> Политики -> Административные шаблоны -> Система -> Доступ к съемным запоминающим устройствам -> Съемные запоминающие устройства всех классов: Запретить любой доступ (включить)
<br>
New-GPO -Name "camera" | New-GPLink -Target "DC=Oaklet,DC=org"
Конфигурация компьютера -> Политики -> Административные шаблоны -> Компоненты Windows -> Камера -> Разрешить использование камер (Отключить)
<br>
New-GPO -Name "desktop" | New-GPLink -Target "DC=Oaklet,DC=org"
Конфигурация пользователя -> Политики -> Административные шаблоны -> Панель управления -> Персонализация
<br>
powershell
gpupdate /force
</pre>

<ul>
    <li><strong>Для обеспечения отказоустойчивости сервер контроллера домена должен выступать DHCP failover для подсети Clients:</strong></li>
    <ul>
        <li>Он должен принимать управление в случае отказа основного DHCP сервера;</li>
    </ul>
</ul>

<pre>
Install-WindowsFeature DHCP –IncludeManagementTools
Set-ItemProperty -Path HKLM:\SOFTWARE\Microsoft\ServerManager\Roles\12 -Name ConfigurationState -Value 2
Restart-Service -Name DHCPServer -Force
<br>
Add-DhcpServerv4Scope -Name “Clients-failover” -StartRange 172.20.2.1 -EndRange 172.20.3.254 -SubnetMask 255.255.254.0 -State InActive
Set-DhcpServerv4OptionValue -ScopeID 172.20.2.0 -DnsDomain Oaklet.org -DnsServer 172.20.0.100,77.88.8.8 -Router 172.20.2.1
Add-DhcpServerv4ExclusionRange -ScopeID 172.20.2.0 -StartRange 172.20.2.1 -EndRange 172.20.2.1
Add-DhcpServerv4ExclusionRange -ScopeID 172.20.2.0 -StartRange 172.20.3.100 -EndRange 172.20.3.100
Set-DhcpServerv4Scope -ScopeID 172.20.2.0 -State Active
</pre>

<ul>
    <li><strong>Организуйте DHCP сервер на базе SRV</strong></li>
    <ul>
        <li>Используйте подсеть Clients учётом существующей инфраструктуры в таблице адресации;</li>
        <li>Клиенты CLI-L и CLI-W получают адрес и все необходимые сетевые параметры по DHCP, обеспечивая связность с сетью Интернет и подсетью Servers;</li>
    </ul>
</ul>
<p>Через веб-интерфейс "https://localhost:8080": (вариант для девочек) </p>

![Image alt](https://github.com/NewErr0r/Qualification_Exam/blob/main/dhcp-web.png)

<p>Вариант для нормальных пацанов:</p>
<pre>
apt-get install -y dhcp-server
</pre>
<pre>
vi /etc/dhcp/dhcpd.conf<br>
ddns-update-style none;
subnet 172.20.2.0 netmask 255.255.254.0 {
        option routers                  172.20.2.1;
        option subnet-mask              255.255.254.0;
        option domain-name              "Oaklet.org";
        option domain-name-servers      172.20.0.100, 77.88.8.8;<br>
        range dynamic-bootp 172.20.3.101 172.20.3.254;
        default-lease-time 21600;
        max-lease-time 43200;
}
</pre>
<pre>
vi /etc/sysconfig/dhcpd<br>
    DHCPDARGS=enp0s3
</pre>
<pre>
systemctl enable --now dhcpd
</pre>

<ul>
    <li><strong>Организуйте сервер времени на базе SRV</strong></li>
    <ul>
        <li>Данный сервер должен использоваться всеми ВМ внутри региона Office;</li>
        <li>Сервер считает собственный источник времени верным;</li>
    </ul>
</ul>

<pre>
apt-get install -y chrony

vi /etc/chrony.conf
    allow 172.20.0.0/24
    allow 172.20.2.0/23
    
systemctl enable --now chronyd
</pre>

<p><strong>DC, FS</p></strong>
<pre>
Start-Service W32Time
w32tm /config /manualpeerlist:172.20.3.100 /syncfromflags:manual /reliable:yes /update
Restart-Service W32Time
</pre>

<p><strong>CLI-W</p></strong>
<pre>
New-NetFirewallRule -DisplayName "NTP" -Direction Inbound -LocalPort 123 -Protocol UDP -Action Allow
</pre>
<pre>
Start-Service W32Time
w32tm /config /manualpeerlist:172.20.3.100 /syncfromflags:manual /reliable:yes /update
Restart-Service W32Time
</pre>
<pre>
Set-Service -Name W32Time -StartupType Automatic
</pre>

<p><strong>CLI-L</p></strong>
<pre>
su -
vi /etc/chrony.conf
    pool 172.20.3.100 iburst
    allow 172.20.2.0/23
    
systemctl restart chronyd
</pre>

<p><strong>FW</p></strong>
<pre>
configure
set system ntp server 172.20.3.100
commit
save
</pre>

<ul>
    <li><strong>Все клиенты региона Office должны быть включены в домен</strong></li>
    <ul>
        <li>С клиентов должен быть возможен вход под любой учётной записью домена;</li>
        <li>На клиентах должны применятся настроенные групповые политики;</li>
        <li>Необходимо обеспечить хранение перемещаемого профиля пользователя Morgushko;</li>
    </ul>
</ul>
<p><strong>CLI-W</p></strong>
<pre>
Rename-Computer -NewName CLI-W
Restart-Computer
</pre>
<pre>
Add-Computer
    Администратор
    P@ssw0rd
        Oaklet.org
Restart-Computer
</pre>

<p><strong>CLI-L</p></strong>
<pre>
su -
hostnamectl set-hostname CLI-L.Oaklet.org
reboot
</pre>
<pre>
su - 
apt-get update
apt-get install -y task-auth-ad-sssd
system-auth write ad Oaklet.org CLI-L Oaklet 'Администратор' 'P@ssw0rd'
reboot
</pre>

<p><strong>DC</p></strong>
<pre>
New-Item -Path "С:\" -Name "roaming_users" -ItemType "directory"<br>
New-SmbShare -Name "roaming_users" -Path "C:\roaming_users\" -FullAccess Oaklet\Администратор
Средства -> Пользователи и компьютеры Active Directory -> Development -> Morgushko (ПКМ) -> Свойства -> Профиль -> Пусть к профилю: \\DC\roaming_users\%username%
</pre>

<ul>
    <li><strong>Организуйте общий каталог для ВМ CLI-W и CLI-L на базе FS:</strong></li>
    <ul>
        <li>Хранение файлов осуществляется на диске, реализованном по технологии RAID5;</li>
        <li>Создать общую папку для пользователей;</li>
        <li>Публикуемый каталог D:\opt\share;</li>
        <li>Смонтируйте каталог на клиентах /mnt/adminshare и D:\adminshare соответственно;</li>
        <li>Разрешите чтение и запись на всех клиентах:
        <ul><li>Определить квоту максимальный размер в 20 мб для пользователей домена;</li></ul>
        </li>
        <li>Монтирование каталогов должно происходить автоматически;</li>        
    </ul>
</ul>

<pre>
diskpart

select disk 1
attrib disk clear readonly
convert dynamic

select disk 2
attrib disk clear readonly
convert dynamic

select disk 3
attrib disk clear readonly
convert dynamic

select disk 4
attrib disk clear readonly
convert dynamic

select disk 5
attrib disk clear readonly
convert dynamic

create volume raid disk=1,2,3,4,5

select volume 0
assign letter=B

select volume 3
assign letter=D
format fs=ntfs
</pre>

<pre>
powershell
Install-WindowsFeature -Name "FS-FileServer"
Install-WindowsFeature -Name "FS-Resource-Manager"
Restart-Computer
</pre>
<pre>
New-Item -Path "D:\" -Name "opt" -ItemType "directory"
New-Item -Path "D:\opt\" -Name "share" -ItemType "directory"<br>
New-SmbShare -Name "share" -Path "D:\opt\share\" -FullAccess Oaklet\Администратор
Grant-SmbShareAccess -Name "share" -AccountName "Oaklet\smb" -AccessRight Full -Force<br>
New-FsrmQuotaTemplate -Name "20MB" -Size 20MB
New-FsrmQuota -Path "D:\opt\share\" -Template "20MB"
</pre>

<p><strong>CLI-W</strong></p>
<pre>
powershell
diskpart
list volume 0 
assign letter=B
</pre>
<pre>
New-SmbMapping -LocalPath D: -RemotePath \\FS\share -Username smb -Password P@ssw0rd -Persistent $true<br>
</pre>

<p><strong>DC</strong></p>
<pre>
GPO:
Конфигурация пользователя -> Настройка -> Конфигурация Windows -> Сопоставление дисков -> ПКМ -> Создать -> Сопоставленный диск ->:
    Размещение: \\FS\share
    Подпись: adminshare
    Использовать: D
-> Общие параметры:
    Выполнять в контексте безопастности вошедшего пользователя
    Нацеливание на уровень элемента -> Нацеливание:
        Создать элемент -> Группа безопасности -> добавить группы<pre>
gpupdate /force   
</pre></pre>

<p><strong>CLI-W</strong></p>
<pre>
powershell
gpupdate /force
</pre>

<p><strong>CLI-L</strong></p>
<pre>
su -
apt-get install -y cifs-utils<br>
mkdir /mnt/adminshare
chmod 777 /mnt/adminshare<br>

echo '//FS.Oaklet.org/share /mnt/adminshare cifs users,credentials=/etc/samba/sabmacreds,file_mode=0777,dir_mode=0777 0 0' >> /etc/fstab<br>
vi /etc/samba/sabmacreds
    username=smb
    password=P@ssw0rd<br>
chmod 600 /etc/samba/sabmacreds
chown root: /etc/samba/sabmacreds<br>
mount -a
</pre>


<ul>
    <li><strong>На файловом сервере FS также хранятся стартовые страницы корпоротивного портала:</strong></li>
    <ul>
        <li>для APP-L  находится  в C:\site\index1.html для APP-L;</li>
        <li>для APP-R  находится в C:\site\index2.html для APP-R;</li>
        <li>Необходимо их загрузить на серверы региона Application вместе с сопутствующими файлами.</li>     
    </ul>
</ul>

<p><strong>APP-V</strong></p>
<pre>
vi /etc/openssh/sshd_config<br>
    PermitRootLogin yes
</pre>
<pre>
systemctl restard sshd.service
</pre>

<p><strong>APP-L</strong></p>
<pre>
vi /etc/openssh/sshd_config<br>
    PermitRootLogin yes
</pre>
<pre>
systemctl restard sshd.service
</pre>

<p><strong>APP-R</strong></p>
<pre>
vi /etc/openssh/sshd_config<br>
    PermitRootLogin yes
</pre>
<pre>
systemctl restard sshd.service
</pre>

<p><strong>FS</strong></p>
<pre>
Add-WindowsCapability -Online -Name OpenSSH.Client*<br>
scp "C:\site\index1.html" "C:\site\index2.html" root@app.first:/tmp
    yes
    P@ssw0rd
</pre

<p><strong>APP-V</strong></p>
<pre>
scp /tmp/index1.html root@10.116.0.20:/tmp
    yes
    P@ssw0rd
</pre>

<pre>
scp /tmp/index2.html root@10.116.0.30:/tmp
    yes
    P@ssw0rd
</pre>

<ul>
    <li><strong>Реализуйте центр сертификации на базе SRV.</strong></li>
    <ul>
        <li>Клиенты СLI-L, CLI-W, CLI-R должны доверять сертификатам;</li>
    </ul>
</ul>

<p><strong>SRV</strong></p>
<p>Через веб-интерфейс "https://localhost:8080": (вариант для девочек) </p>

![Image alt](https://github.com/NewErr0r/Qualification_Exam/blob/main/CA.png)

<p>Вариант для нормальных пацанов:</p>
<pre>
su - 
mkdir /var/ca
cd /var/ca<br>
openssl req -newkey rsa:4096 -keyform PEM -keyout ca.key -x509 -days 3650 -outform PEM -out ca.cer
    P@ssw0rd
    P@ssw0rd
    Country Name: RU
    Organization Name: Oaklet.org
    Common Name: Oaklet.org CA
</pre>
<pre>
vi /etc/openssh/sshd_config<br>
    PermitRootLogin yes
</pre>
<pre>
systemctl restart sshd.service
</pre>

<p><strong>FS</strong></p>
<pre>
scp root@SRV.Oaklet.org:/var/ca/ca.cer D:\opt\share
    yes
    P@ssw0rd
</pre>

<p><strong>CLI-W</strong></p>
<pre>
Import-Certificate -FilePath "D:\ca.cer" -CertStoreLocation cert:\CurrentUser\Root
</pre>

<p><strong>CLI-L</strong></p>
<pre>
su -
cp /mnt/adminshare/ca.cer /etc/pki/ca-trust/source/anchors/ && update-ca-trust
</pre>

<p><strong>CLI-R</strong></p>
<pre>
ip route add 172.20.0.0/24 via 200.100.100.100
ip route add 172.20.2.0/23 via 200.100.100.100<br>
scp root@SRV.Oaklet.org:/var/ca/ca.cer /etc/pki/ca-trust/source/anchors/ && update-ca-trust
    yes
    P@ssw0rd
</pre>

<ul>
    <li><strong>Необходимо реализовать следующую инфраструктуру приложения на базе SRV</strong></li>
    <ul>
        <li>На нём должно быть активировано внутренне приложение, исполняющееся в контейнере и отвечающее на запросы из браузера клиентов (образ и все необходимые пакеты для работы приложения уже установленны);</li>
        <li>Образ приложения расположен по пути: /home/admin/docker;</li>
        <li>Доступ к приложению осуществляется по DNS-имени www.Oaklet.org;</li>     
    </ul>
</ul>

<p><strong>SRV</strong></p>
<pre>
su -<br>
systemctl enable --now docker.service<br>
cd /home/admin/docker<br>
docker build -t  app .
docker run --name app -p 80:5000 -d app
</pre

![Image alt](https://github.com/NewErr0r/Qualification_Exam/blob/main/docker.png)

<ul>
    <li><strong>На клиенте под управлением Windows должен быть создан шифрованный Bitlocker раздел диска от пользователя уровня ADM.</strong></li>
</ul>

<p><strong>CLI-W</strong></p>
<pre>
powershell
logoff
    CLI-W\user
    P@ssw0rd
</pre>
<pre>
Управление дисками -> Диск 0 -> ПКМ -> Сжать том -> Размер сжимаемого пространства: 5120 -> Сжать
ПКМ -> Создать простой том -> U:
</pre>
<pre>
logoff
    Director
    P@ssw0rd<br>
# Должен быть в группе Администраторы Домена
Enable-BitLocker -MountPoint U: -PasswordProtector
</pre>

<ul>
    <li><strong>В общих локальных документах должен быть создан текстовый файл с записью внутри: “Секретное содержимое”, зашифрованный EFS.</strong></li>
</ul>
<p><strong>CLI-W</strong></p>

<pre>
New-Item -Path "C:\Users\Public\Documents\file.txt" -ItemType "file" -Value "Секретное содержимое"<br>
cipher /e C:\Users\Public\Documents\file.txt
</pre>

<h1>Сетевая связность:</h1>
<ul>
    <li><strong>Реализуйте связность конпонентов инфраструктуры с применением технологии VPN.</strong></li>
        <ul>
        <li>Соединяются регионы Office и Application;</li>
        <li>Соединение должно быть защищено;</li>
        <li>При повторном запуске – восстанавливаться;</li>
        <li>Все хосты регионов должны взаимодействовать друг с другом;</li>
    </ul>
</ul>
<p><strong>FW</strong></p>
<pre>
sudo su
mkdir /etc/wireguard/keys
cd /etc/wireguard/keys
</pre>
<pre>
wg genkey | tee srv-sec.key | wg pubkey > srv-pub.key
wg genkey | tee cli-sec.key | wg pubkey > cli-pub.key<br>
cat srv-sec.key cli-pub.key >> /etc/wireguard/wg0.conf<br>
vi /etc/wireguard/wg0.conf
  [Interface]
  Address = 10.20.30.1/30
  ListenPort = 12345
  PrivateKey = srv-sec.key<br>  
  [Peer]
  PublicKey = cli-pub.key
  AllowedIPs = 10.20.30.0/30<br>
systemctl enable --now wg-quick@wg0<br>
scp srv-pub.key cli-sec.key root@200.100.100.200:/tmp
reboot
</pre>

<p><strong>APP-V</strong></p>
<pre>
apt-get install -y wireguard-tools wireguard-tools-wg-quick
</pre>
<pre>
cd /tmp<br>
mkdir /etc/wireguard<br>
cat cli-sec.key srv-pub.key >> /etc/wireguard/wg0.conf<br>
vi /etc/wireguard/wg0.conf
  [Interface]
  Address = 10.20.30.2/30
  PrivateKey = cli-sec.key<br>
  [Peer]
  PublicKey = srv-pub.key
  Endpoint = 200.100.100.100:12345
  AllowedIPs = 10.20.30.0/30, 172.20.0.0/24, 172.20.2.0/23
  PersistentKeepalive = 10
</pre>
<pre>
firewall-cmd --permanent --add-port=12345/{tcp,udp}
firewall-cmd --permanent --add-interface=wg0 --zone=public
firewall-cmd --reload
</pre>
<pre>
systemctl enable --now wg-quick@wg0
</pre>

<ul>
    <li><strong>CLI-R должен получать удалённый доступ к каналам инфраструктуры, в частности к DNS серверу.</strong></li>
</ul>
<p><strong>Выполнен ранее</strong></p>

<h1>Работа приложения:</h1>
<ul>
    <li><strong>В регионе APP должен хостится корпоративный портал. Он развёртывается на APP-L и APP-R. </strong></li>
</ul>
<p><strong>APP-L</strong></p>
<pre>
apt-get update
apt-get install -y nginx<br>
systemctl enable --now nginx<br>
mkdir -p /var/www/html
cp /opt/index1.html /var/www/html<br>
mv /var/www/html/index1.html /var/www/html/index.html
</pre>
<pre>
vi /etc/nginx/sites-available.d/default.conf
    listen 80 default_server;<br>
ln -s /etc/nginx/sites-available.d/default.conf /etc/nginx/sites-enabled.d/default.conf
</pre>
<pre>
nginx -t 
systemctl restart nginx
</pre>

<p><strong>APP-R</strong></p>
<pre>
apt-get update
apt-get install -y nginx<br>
systemctl enable --now nginx<br>
mkdir -p /var/www/html
cp /opt/index2.html /var/www/html<br>
mv /var/www/html/index2.html /var/www/html/index.html
</pre>
<pre>
vi /etc/nginx/sites-available.d/default.conf
    listen 80 default_server;<br>
ln -s /etc/nginx/sites-available.d/default.conf /etc/nginx/sites-enabled.d/default.conf
</pre>
<pre>
nginx -t 
systemctl restart nginx
</pre>

<ul>
    <li><strong>Корпоративный портал должен быть доступен по адресу app.first.</strong></li>
        <ul>
        <li>Клиентами должны быть CLI-L, CLI-W, CLI-R.</li>
        <li>Доступ должен осуществляться по внешнему каналу, внутренний прямой доступ к вебслужбам на на хостах APP-L и APP-R запрещён.</li>
        <li>Доступ к порталу должен осуществляться по защищённому каналу. Незащищённые HTTP-соединения автоматически переводятся на защищённый канал. Перевод осуществляется с сохранением параметров запроса.</li>
    </ul>
    <li><strong>В любом из сценариев высокой доступности простой не должен составлять более 20 секунд. </strong></li>
    <li>Портал должен быть доступен при отказе одного из APP-L(R) хостов.</strong></li>
</ul>
<p><strong>APP-V</strong></p>

<pre>
mkdir cert
cd cert<br>
openssl genrsa -out app.key 4096
openssl req -new -key app.key -out app.req -sha256
    Country Name: RU
    Organization Name: Oaklet.org
    Common Name: app.first
</pre>
<pre>
scp app.req root@172.20.3.100:/var/ca
</pre>

<p><strong>SRV</strong></p>
<pre>
su -
cd /var/ca
</pre>
<pre>
openssl x509 -req -in app.req -CA ca.cer -CAkey ca.key -set_serial 100 -extentions app -days 1460 -outform PEM -out app.cer -sha256
    P@ssw0rd
</pre>

<p><strong>APP-V</strong></p>
<pre>
scp root@172.20.3.100:/var/ca/app.cer ./
</pre>
<pre>
mkdir -p /etc/pki/nginx/private<br>
cp app.cer /etc/pki/nginx/
cp app.key /etc/pki/nginx/private
</pre>
<pre>
apt-get install -y nginx
systemctl enable --now nginx<br>
vi /etc/nginx/sites-available.d/proxy.conf
</pre>

![Image alt](https://github.com/NewErr0r/Qualification_Exam/blob/main/proxy_conf.png)

<pre>
ln -s /etc/nginx/sites-available.d/proxy.conf /etc/nginx/sites-enabled.d/proxy.conf<br>
nginx -t
systemctl restart nginx 
</pre>
<pre>
firewall-cmd --permanent --add-service={http,https}
firewall-cmd --reload
</pre>

<p><strong>CLI-W</strong></p>

![Image alt](https://github.com/NewErr0r/Qualification_Exam/blob/main/index1_html.png)

<p><strong>CLI-L</strong></p>

![Image alt](https://github.com/NewErr0r/Qualification_Exam/blob/main/index2_html.png)
