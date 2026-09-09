# Задание 1
Описание
Изучен проект. Выполнена инициализация Terraform и применен код. Создана группа безопасности.

Выполненные шаги
Создана директория 03/src/

Скопированы базовые файлы из предыдущего ДЗ

Создан файл security-group.tf с группой безопасности для web ВМ:

resource "yandex_vpc_security_group" "web_sg" {

  name        = "web-security-group"
  
  description = "Security group for web VMs"
  
  network_id  = yandex_vpc_network.network.id

  ingress {
  
  protocol       = "TCP"
    
  description    = "HTTP"
    
  v4_cidr_blocks = ["0.0.0.0/0"]
    
  port           = 80
    
  }

  ingress {
  
  protocol       = "TCP"
    
  description    = "HTTPS"
    
  v4_cidr_blocks = ["0.0.0.0/0"]
    
  port           = 443
    
  }

  ingress {
  
  protocol       = "TCP"
    
  description    = "SSH"
    
  v4_cidr_blocks = ["0.0.0.0/0"]
    
  port           = 22
    
  }

  egress {
  
  protocol       = "ANY"
    
  description    = "Allow all outgoing"
    
  v4_cidr_blocks = ["0.0.0.0/0"]
    
  }
  
}

Создана сеть и подсеть:

resource "yandex_vpc_network" "network" {

  name = "netology-network-03"
  
}

resource "yandex_vpc_subnet" "subnet" {

  name           = "netology-subnet-03"
  
  zone           = "ru-central1-a"
  
  network_id     = yandex_vpc_network.network.id
  
  v4_cidr_blocks = ["10.0.0.0/24"]
  
}

Выполнена инициализация и применение:

terraform init

terraform apply -auto-approve

Скриншот
Скриншот 1: Входящие правила группы безопасности в ЛК Yandex Cloud

![Terrafor](Terraform.png)

На скриншоте видны входящие правила: HTTP (порт 80), HTTPS (порт 443), SSH (порт 22) с разрешением от 0.0.0.0/0.

# Задание 2
Описание
Созданы 4 ВМ с использованием мета-аргументов count и for_each. Web ВМ используют группу безопасности из Задания 1. SSH-ключ считывается из файла через функцию file в local-переменной.

Выполненные шаги
1. Чтение SSH-ключа в locals.tf:

hcl
locals {
  ssh_public_key = file("~/.ssh/id_rsa.pub")
}
2. Создание ВМ баз данных через for_each (for_each-vm.tf):

hcl
variable "each_vm" {
  type = list(object({
    vm_name     = string
    cpu         = number
    ram         = number
    disk_volume = number
  }))
  default = [
    {
      vm_name     = "main"
      cpu         = 2
      ram         = 4
      disk_volume = 20
    },
    {
      vm_name     = "replica"
      cpu         = 2
      ram         = 2
      disk_volume = 10
    }
  ]
}

resource "yandex_compute_instance" "db" {
  for_each = { for vm in var.each_vm : vm.vm_name => vm }
  
  name        = each.value.vm_name
  platform_id = "standard-v1"
  
  resources {
    cores  = each.value.cpu
    memory = each.value.ram
  }
  
  boot_disk {
    initialize_params {
      image_id = "fd8d6s0blceqbto92ss8"
      size     = each.value.disk_volume
    }
  }
  
  network_interface {
    subnet_id = yandex_vpc_subnet.subnet.id
    nat       = true
  }
  
  metadata = {
    ssh-keys = "ubuntu:${local.ssh_public_key}"
  }
  
  scheduling_policy {
    preemptible = true
  }
}
3. Создание web ВМ через count (count-vm.tf):

hcl
resource "yandex_compute_instance" "web" {
  count = 2
  
  name        = "web-${count.index + 1}"
  platform_id = "standard-v1"
  
  resources {
    cores  = 2
    memory = 1
  }
  
  boot_disk {
    initialize_params {
      image_id = "fd8d6s0blceqbto92ss8"
      size     = 10
    }
  }
  
  network_interface {
    subnet_id          = yandex_vpc_subnet.subnet.id
    nat                = true
    security_group_ids = [yandex_vpc_security_group.web_sg.id]
  }
  
  metadata = {
    ssh-keys = "ubuntu:${local.ssh_public_key}"
  }
  
  scheduling_policy {
    preemptible = true
  }
  
  depends_on = [yandex_compute_instance.db]
}
4. Применены изменения:

bash
terraform apply -auto-approve
Результат
Созданы ВМ:

main (2 vCPU, 4 GB RAM)

replica (2 vCPU, 2 GB RAM)

web-1 (2 vCPU, 1 GB RAM)

web-2 (2 vCPU, 1 GB RAM)

# Задание 3
Описание
Созданы 3 диска по 1 ГБ с использованием count. Создана ВМ storage с подключением всех 3 дисков через dynamic secondary_disk с for_each.

Выполненные шаги
Файл disk_vm.tf:

hcl
# Создание 3 дисков
resource "yandex_compute_disk" "storage_disk" {
  count = 3
  
  name     = "storage-disk-${count.index + 1}"
  type     = "network-hdd"
  size     = 1
  zone     = "ru-central1-a"
}

# Создание ВМ storage
resource "yandex_compute_instance" "storage" {
  name        = "storage"
  platform_id = "standard-v1"
  
  resources {
    cores  = 2
    memory = 2
  }
  
  boot_disk {
    initialize_params {
      image_id = "fd8d6s0blceqbto92ss8"
      size     = 10
    }
  }
  
  network_interface {
    subnet_id = yandex_vpc_subnet.subnet.id
    nat       = true
  }
  
  dynamic "secondary_disk" {
    for_each = yandex_compute_disk.storage_disk[*].id
    content {
      disk_id = secondary_disk.value
    }
  }
  
  metadata = {
    ssh-keys = "ubuntu:${local.ssh_public_key}"
  }
  
  scheduling_policy {
    preemptible = true
  }
}
Скриншот
Скриншот 2: ВМ storage с тремя подключенными дисками

https://screenshots/storage-disks.png

На скриншоте видна ВМ storage и три дополнительных диска: storage-disk-1, storage-disk-2, storage-disk-3.

# Задание 4
Описание
Создан inventory-файл для Ansible с помощью функции templatefile. Инвентарь содержит 3 группы: webservers, databases, storage. Добавлена переменная fqdn.

Выполненные шаги
1. Файл-шаблон inventory.tpl:

jinja
[webservers]
%{ for vm in web_vms ~}
${vm.name} ansible_host=${vm.external_ip} fqdn=${vm.fqdn}
%{ endfor ~}

[databases]
%{ for vm in db_vms ~}
${vm.name} ansible_host=${vm.external_ip} fqdn=${vm.fqdn}
%{ endfor ~}

[storage]
${storage_vm.name} ansible_host=${storage_vm.external_ip} fqdn=${storage_vm.fqdn}
2. Файл ansible.tf:

hcl
locals {
  inventory = templatefile("${path.module}/inventory.tpl", {
    web_vms = [
      for i, vm in yandex_compute_instance.web : {
        name        = vm.name
        external_ip = vm.network_interface[0].nat_ip_address
        fqdn        = vm.fqdn
      }
    ]
    db_vms = [
      for name, vm in yandex_compute_instance.db : {
        name        = name
        external_ip = vm.network_interface[0].nat_ip_address
        fqdn        = vm.fqdn
      }
    ]
    storage_vm = {
      name        = yandex_compute_instance.storage.name
      external_ip = yandex_compute_instance.storage.network_interface[0].nat_ip_address
      fqdn        = yandex_compute_instance.storage.fqdn
    }
  })
}

resource "local_file" "inventory" {
  filename = "${path.module}/inventory.ini"
  content  = local.inventory
}
3. Применены изменения и создан файл inventory.ini.

Скриншот
Скриншот 3: Файл inventory.ini

https://screenshots/inventory-file.png

На скриншоте видны группы webservers (web-1, web-2), databases (main, replica), storage (storage) с указанием ansible_host и fqdn.
