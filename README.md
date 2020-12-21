# Install Ubuntu + Docker Swarm + Moodle

## periksa seluruh interface NIC
```bash
ifconfig -a
```

## konfig LAN
```bash
nano /etc/netplan/00-installer-config.yaml
```
### terus masukin syntax yaml sesuai kebutuhan dan topologi di sekolah
contoh di jaringan saya, berikut ini
```yaml
network:
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      addresses: [172.16.10.11/24]
  version: 2
```

### Ubah Hostname di Ubuntu
```bash
hostnamectl set-hostname Master
```
### Ubah timezone WIB
```bash
timedatectl set-timezone Asia/Jakarta
```
### ===================
## instalasi docker
```bash
apt update -y && apt upgrade -y
```
enter
```bash
apt install docker.io -y
```
enter dan tunggu proses instalasinya,
jalankan docker
```bash
systemctl start docker
```
enter, 
aktifkan docker saat booting
```bash
systemctl enable docker
```
cek status docker
```bash
systemctl status docker
```
## Docker Swarm

### Inisialisasi docker swarm pada node 1
```bash
root@node1:~# docker swarm init --advertise-addr [ip private node1]
```
### Melihat join token docker swarm pada node 1 dan join node 2 ke cluster swarm
```bash
root@node1:~# docker swarm join-token worker
```
Copy perintah join swarm pada node 2
```bash
root@node2:~# docker swarm join --token [join token] [ip private node1]:2377
```
### Mengecek anggota cluster swarm
```bash
root@node1:~# docker node ls
```
## Install Portainer
### Download file Portainer
```bash
cd && mkdir moodle
cd moodle
curl -L https://downloads.portainer.io/portainer-agent-stack.yml -o portainer-agent-stack.yml
```
### Menjalankan stack Portainer
```bash
docker stack deploy --compose-file=portainer-agent-stack.yml portainer
docker stack ls
docker stack services portainer
```
Akses lewat browser menggunakan ip node1:9000
Atur username dan password

## Studi Kasus Moodle dengan Docker Swarm
### Prasyarat
Sebelum memulai, pastikan database yang digunakan sudah terkonfigurasi (non container)

mariadb versi 10.3 ke atas
redis versi terbaru
### Instalasi dan konfigurasi GlusterFS
Glusterfs akan digunakan sebagai volume persistent ketika menjalankan moodle di docker swarm, sehingga ketika moodle di deploy ulang,
data yang sudah ada tidak akan hilang.

Jalankan sebagai user root di semua node
```bash
add-apt-repository ppa:gluster/glusterfs-3.12
```
```bash
apt install glusterfs-server -y
```
```bash
systemctl start glusterd && systemctl enable glusterd
```

Generate ssh key baru untuk setiap node (masih dengan user root)
```bash
ssh-keygen -t rsa
```

Sebelum melakukan probing gluster node, pastikan IP address setiap node sudah di-mapping ke ```/etc/hosts```
```bash
nano /etc/hosts
```
sebagai contoh:
```
192.168.1.10 docker-node1
192.168.1.20 docker-node2
192.168.1.30 docker-node3
```

Setelah itu, lakukan peer probe pada node 1 atau node yang menjadi docker swarm manager
```bash
gluster peer probe docker-node2
gluster peer probe docker-node3
```
Cek kembali peer pool gluster yang sudah ditambahkan
```bash
gluster pool list
```

Buat direktori baru di semua node
```bash
mkdir -p /gluster/moodle-volume
```

Kemudian buat volume glusterfs, lakukan hanya di node 1 atau node yang menjadi docker swarm manager
```bash
gluster volume create staging-gfs replica 3 docker-node1:/gluster/moodle-volume docker-node2:/gluster/moodle-volume docker-node3:/gluster/moodle-volume force
```
```replica``` disini mengikuti jumlah node yang ada di gluster pool, jika terdapat 3 pool maka replica = 3
Jalankan volume glusterfs
```bash
gluster volume start staging-gfs
```

Selanjutnya, pastikan volume gluster yang sudah dibuat akan ter-mount otomatis ketika server restart, lakukan di semua node
```bash
echo 'localhost:/staging-gfs /mnt/moodle-volume glusterfs defaults,_netdev,backupvolfile-server=localhost 0 0' >> /etc/fstab
```
Mount volume glusterfs
```bash
mount.glusterfs localhost:/staging-gfs /mnt/moodle-volume
```
Ubah hak kepemilikan volume
```bash
chown -R root:docker /mnt/moodle-volume
```

Cek kembali apakah volume glusterfs sudah ter-mount
```bash
df -h | grep /mnt/moodle-volume
```

### Pada portainer, buka Menu Portainer > Stack dan masukkan konfigurasi docker compose berikut
```yaml
version: '3.8'
services:
  moodle:
    image: 'azemoning/moodle-redis'
    ports:
      - '80:8080'
      - '443:8443'
    environment:
      - MOODLE_SKIP_INSTALL=yes
      - MOODLE_DATABASE_HOST=mariadbhost
      - MOODLE_DATABASE_PORT_NUMBER=3306
      - MOODLE_DATABASE_USER=dbuser
      - MOODLE_DATABASE_NAME=dbname
      - MOODLE_DATABASE_PASSWORD=mariadbpassword
      - BITNAMI_DEBUG=true
      - TZ=Asia/Jakarta
    volumes:
      - type: bind
        source: /mnt/moodle-volume/moodle
        destination: /bitnami/moodle
      - type: bind
        source: /mnt/moodle-volume/moodledata
        destination: /bitnami/moodledata
```
#### klik tombol Deploy the stack
#### ..tunggu sampai proses selesai
#### lalu buka browser http://ip_node1

```
