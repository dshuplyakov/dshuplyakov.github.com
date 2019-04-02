---
title: Быстрая установка Kubernetes с админкой на RHEL
author: Дмитрий Шупляков
tags: Kubernetes
date: 2019-03-31 13:54:00
---

Инструкция для установки кубернетиса на одной мастер-ноде, с возможностью запуска на ней pod'ов.
<!-- more --> 

Первым дело обновляем пакеты и репозитории:
```bash
sudo su
yum upgrade -y
```

Устанавливаем docker
```bash
yum install -y docker
```
Добавляем докер в автозапуск:
```bash
systemctl enable docker 
```
Cgroups у докера и кубернетиса должны совпадать. Кубернетис по умолчанию запукается из под cgroupsfs, а докер из под systemd. Меняем у докера, для этого в файле /usr/lib/systemd/system/docker.service ищем строку:

```bash
--exec-opt native.cgroupdriver=systemd \
```
и заменяем systemd на cgroupfs.

Проверяем, что изменения применились:
```bash
docker info
```
В выводе должна быть строчка:
```bash
Cgroup Driver: cgroupfs
```

Для работы кубернетиса необходимо выключить своп:
```bash
swapoff -a
```
И не забываем удалить строчку с подключением свопа из /etc/fstab

Отлично! Начинаем ставим кубик:
```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
setenforce 0
yum install -y kubelet kubeadm kubectl
systemctl enable kubelet && systemctl start kubelet
```

У меня установились следующие версии:
Installed:
  kubeadm.x86_64 0:1.14.0-0                       kubectl.x86_64 0:1.14.0-0                       kubelet.x86_64 0:1.14.0-0

Проверяем, смогла запуститься служба kubelet `systemctl status kubelet`
Если нет, то запускаем вручную и смотрим, какие ошибки:
```bash
kubelet
```

Далее необходимо выйти из под рута, и под обычным пользователем выполнить:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Этой командой конфиг кубика копируется в личную папку пользователя в файл config.

Разрешаем подам запускаться на мастер-ноде:
```bash
kubectl taint nodes --all node-role.kubernetes.io/master- 
```

## Установка админки

Скачиваем файл:
```bash
wget https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended/kubernetes-dashboard.yaml
```
Редактируем секцию args, добавляя строки в файл kubernetes-dashboard.yaml:
```bash
          - --enable-skip-login
          - --enable-insecure-login
```
Запускаем pod:
```bash
kubectl create -f kubernetes-dashboard.yaml
```

Меняем тип pod'a, как это сделать - читаем здесь https://github.com/kubernetes/dashboard/wiki/Accessing-Dashboard---1.7.X-and-above

И запускаем прокси, чтобы можно было снаружи зайти в админку (вместо 172.30.179.137 подставляем свой ip-адрес):
```bash
kubectl proxy --accept-hosts='^.*' --address=172.30.179.137
```

Теперь админка доступна по адресу:
[http://172.30.179.137:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login](http://172.30.179.137:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login)

Чтобы зайти внутрь, нажимаем кнопку *SKIP*.

