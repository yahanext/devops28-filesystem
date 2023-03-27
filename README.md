# Домашнее задание к занятию «Файловые системы»

### Цель задания

В результате выполнения задания вы: 

* научитесь работать с инструментами разметки жёстких дисков, виртуальных разделов — RAID-массивами и логическими томами, конфигурациями файловых систем. Основная задача — понять, какие слои абстракций могут нас отделять от файловой системы до железа. Обычно инженер инфраструктуры не сталкивается напрямую с настройкой LVM или RAID, но иметь понимание, как это работает, необходимо;
* создадите нештатную ситуацию работы жёстких дисков и поймёте, как система RAID обеспечивает отказоустойчивую работу.


### Чеклист готовности к домашнему заданию

1. Убедитесь, что у вас на новой виртуальной машине  установлены утилиты: `mdadm`, `fdisk`, `sfdisk`, `mkfs`, `lsblk`, `wget` (шаг 3 в задании).  
2. Воспользуйтесь пакетным менеджером apt для установки необходимых инструментов.


### Дополнительные материалы для выполнения задания

1. Разряженные файлы — [sparse](https://ru.wikipedia.org/wiki/%D0%A0%D0%B0%D0%B7%D1%80%D0%B5%D0%B6%D1%91%D0%BD%D0%BD%D1%8B%D0%B9_%D1%84%D0%B0%D0%B9%D0%BB).
2. [Подробный анализ производительности RAID](https://www.baarf.dk/BAARF/0.Millsap1996.08.21-VLDB.pdf), страницы 3–19.
3. [RAID5 write hole](https://www.intel.com/content/www/us/en/support/articles/000057368/memory-and-storage.html).


------

## Задание

1. Узнайте о [sparse-файлах](https://ru.wikipedia.org/wiki/%D0%A0%D0%B0%D0%B7%D1%80%D0%B5%D0%B6%D1%91%D0%BD%D0%BD%D1%8B%D0%B9_%D1%84%D0%B0%D0%B9%D0%BB) (разряженных).

1. Могут ли файлы, являющиеся жёсткой ссылкой на один объект, иметь разные права доступа и владельца? Почему?
```
Нет не могут, жесткая ссылка и файл имеют одинаковый индексный дескриптор, по сути жесткая ссылка указатель на inod.
```

1. Сделайте `vagrant destroy` на имеющийся инстанс Ubuntu. Замените содержимое Vagrantfile следующим:

    ```ruby
    path_to_disk_folder = './disks'

    host_params = {
        'disk_size' => 2560,
        'disks'=>[1, 2],
        'cpus'=>2,
        'memory'=>2048,
        'hostname'=>'sysadm-fs',
        'vm_name'=>'sysadm-fs'
    }
    Vagrant.configure("2") do |config|
        config.vm.box = "bento/ubuntu-20.04"
        config.vm.hostname=host_params['hostname']
        config.vm.provider :virtualbox do |v|

            v.name=host_params['vm_name']
            v.cpus=host_params['cpus']
            v.memory=host_params['memory']

            host_params['disks'].each do |disk|
                file_to_disk=path_to_disk_folder+'/disk'+disk.to_s+'.vdi'
                unless File.exist?(file_to_disk)
                    v.customize ['createmedium', '--filename', file_to_disk, '--size', host_params['disk_size']]
                    v.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', disk.to_s, '--device', 0, '--type', 'hdd', '--medium', file_to_disk]
                end
            end
        end
        config.vm.network "private_network", type: "dhcp"
    end
    ```

    Эта конфигурация создаст новую виртуальную машину с двумя дополнительными неразмеченными дисками по 2,5 Гб.

1. Используя `fdisk`, разбейте первый диск на два раздела: 2 Гб и оставшееся пространство.

1. Используя `sfdisk`, перенесите эту таблицу разделов на второй диск.
```
fdisk создал два раздела  /dev/sdb1 /dev/sdb2
командой sfdisk -d /dev/sdb | sfdisk /dev/sdc перенес таблицу разделов
1. Соберите `mdadm` RAID1 на паре разделов 2 Гб.
root@sysadm-fs:/home/vagrant# mdadm --create --verbose /dev/md0 --level=1  --raid-devices=2 /dev/sdc1 /dev/sdb1


1. Соберите `mdadm` RAID0 на второй паре маленьких разделов.
root@sysadm-fs:/home/vagrant# mdadm --create --verbose /dev/md1 --level=0  --raid-devices=2 /dev/sdc2 /dev/sdb2
root@sysadm-fs:/home/vagrant# mdadm --detail --scan > /etc/mdadm/mdadm.conf



1. Создайте два независимых PV на получившихся md-устройствах.


1. Создайте общую volume-group на этих двух PV.

1. Создайте LV размером 100 Мб, указав его расположение на PV с RAID0.

1. Создайте `mkfs.ext4` ФС на получившемся LV.

1. Смонтируйте этот раздел в любую директорию, например, `/tmp/new`.

1. Поместите туда тестовый файл, например, `wget https://mirror.yandex.ru/ubuntu/ls-lR.gz -O /tmp/new/test.gz`.

1. Прикрепите вывод `lsblk`.

1. Протестируйте целостность файла:

    ```bash
    root@vagrant:~# gzip -t /tmp/new/test.gz
    root@vagrant:~# echo $?
    0
    ```

1. Используя pvmove, переместите содержимое PV с RAID0 на RAID1.

1. Сделайте `--fail` на устройство в вашем RAID1 md.

1. Подтвердите выводом `dmesg`, что RAID1 работает в деградированном состоянии.

1. Протестируйте целостность файла — он должен быть доступен несмотря на «сбойный» диск:

    ```bash
    root@vagrant:~# gzip -t /tmp/new/test.gz
    root@vagrant:~# echo $?
    0
    ```

1. Погасите тестовый хост — `vagrant destroy`.
 
*В качестве решения пришлите ответы на вопросы и опишите, как они были получены.*

----

### Правила приёма домашнего задания

В личном кабинете отправлена ссылка на .md-файл в вашем репозитории.


### Критерии оценки

Зачёт:

* выполнены все задания;
* ответы даны в развёрнутой форме;
* приложены соответствующие скриншоты и файлы проекта;
* в выполненных заданиях нет противоречий и нарушения логики.

На доработку:

* задание выполнено частично или не выполнено вообще;
* в логике выполнения заданий есть противоречия и существенные недостатки. 
