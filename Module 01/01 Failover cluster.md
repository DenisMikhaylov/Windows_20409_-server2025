
## Лабораторная работа: Развертывание отказоустойчивого кластера Hyper-V на Windows Server 2025 (с использованием разностных дисков)

**Цель работы:** Научиться развертывать и настраивать отказоустойчивый кластер Hyper-V на базе Windows Server 2025, используя iSCSI-хранилище и технологию разностных дисков для экономии места.

**Архитектура:**
*   **Родительский диск:** `base2025.vhdx` — содержит чистую установку Windows Server 2025.
*   **LON-HOST1:** Первый узел кластера (ВМ1) на разностном диске.
*   **LON-HOST2:** Второй узел кластера (ВМ2) на разностном диске.
*   **LON-SS1:** Сервер хранения данных (ВМ3) на разностном диске.

---

### Часть 1: Подготовка родительского диска и создание виртуальных машин

**1.1. Подготовка родительского диска (опционально)**

Убедитесь, что у вас есть родительский виртуальный жесткий диск `base2025.vhdx` с установленной Windows Server 2025 (желательно Datacenter Edition). Выполните на нем **Sysprep** (обобщение), чтобы он был готов к клонированию:

```powershell
# Запустите внутри виртуальной машины с base2025
C:\Windows\System32\Sysprep\sysprep.exe /oobe /generalize /shutdown
```

После завершения Sysprep виртуальная машина выключится. Теперь диск `base2025.vhdx` готов к использованию в качестве родительского. **Важно:** Установите на родительском диске атрибут "Только чтение", чтобы случайно не изменить его:

```powershell
# На физическом хосте Hyper-V
Set-ItemProperty -Path "C:\VHDs\base2025.vhdx" -Name IsReadOnly -Value $true
```

**1.2. Создание разностных дисков для виртуальных машин**

Создайте три разностных диска для каждой виртуальной машины. Разностные диски будут хранить только изменения, внесенные в каждой конкретной ВМ.

```powershell
# Создаем разностный диск для LON-HOST1
New-VHD -Path "C:\VHDs\LON-HOST1.vhdx" -ParentPath "C:\VHDs\base2025.vhdx" -Differencing

# Создаем разностный диск для LON-HOST2
New-VHD -Path "C:\VHDs\LON-HOST2.vhdx" -ParentPath "C:\VHDs\base2025.vhdx" -Differencing

# Создаем разностный диск для LON-SS1
New-VHD -Path "C:\VHDs\LON-SS1.vhdx" -ParentPath "C:\VHDs\base2025.vhdx" -Differencing
```

**1.3. Создание виртуальных машин**

Создайте три виртуальные машины, указав в качестве загрузочного диска соответствующий разностный диск:

```powershell
# Создание LON-HOST1
New-VM -Name "LON-HOST1" -MemoryStartupBytes 4GB -BootDevice VHD -VHDPath "C:\VHDs\LON-HOST1.vhdx" -Path "C:\Hyper-V" -Generation 2 -SwitchName "Внешняя сеть"

# Создание LON-HOST2
New-VM -Name "LON-HOST2" -MemoryStartupBytes 4GB -BootDevice VHD -VHDPath "C:\VHDs\LON-HOST2.vhdx" -Path "C:\Hyper-V" -Generation 2 -SwitchName "Внешняя сеть"

# Создание LON-SS1
New-VM -Name "LON-SS1" -MemoryStartupBytes 2GB -BootDevice VHD -VHDPath "C:\VHDs\LON-SS1.vhdx" -Path "C:\Hyper-V" -Generation 2 -SwitchName "Внешняя сеть"
```

**1.4. Включение вложенной виртуализации (Nested Virtualization)**

Поскольку узлы кластера сами будут виртуальными машинами, им нужно разрешить запускать внутри себя Hyper-V:

```powershell
Set-VMProcessor -VMName "LON-HOST1" -ExposeVirtualizationExtensions $true
Set-VMProcessor -VMName "LON-HOST2" -ExposeVirtualizationExtensions $true
```

**1.5. Настройка параметров виртуальных машин**

Настройте количество процессоров и другие параметры:

```powershell
Set-VMProcessor -VMName "LON-HOST1" -Count 2
Set-VMProcessor -VMName "LON-HOST2" -Count 2
Set-VMProcessor -VMName "LON-SS1" -Count 1

# Включение динамической памяти (опционально)
Set-VMMemory -VMName "LON-HOST1" -DynamicMemoryEnabled $true -MinimumBytes 2GB -MaximumBytes 8GB
Set-VMMemory -VMName "LON-HOST2" -DynamicMemoryEnabled $true -MinimumBytes 2GB -MaximumBytes 8GB
```

**1.6. Настройка сети внутри виртуальных машин**

Запустите все три виртуальные машины и настройте статические IP-адреса:
Посмотреть реальные ИП адреса в DNS на LON-DC1
| Виртуальная машина | IP-адрес | Маска подсети | DNS (опционально) |
|---|---|---|---|
| LON-HOST1 | 192.168.10.1 | 255.255.255.0 | 192.168.10.1 (или свой DNS) |
| LON-HOST2 | 192.168.10.2 | 255.255.255.0 | 192.168.10.1 |
| LON-SS1 | 192.168.10.3 | 255.255.255.0 | 192.168.10.1 |

**Важно:** Переименуйте серверы в соответствии с именами и перезагрузите их:

```powershell
# На каждой ВМ
Rename-Computer -NewName "LON-HOST1" -Restart  # для LON-HOST1
Rename-Computer -NewName "LON-HOST2" -Restart  # для LON-HOST2
Rename-Computer -NewName "LON-SS1" -Restart    # для LON-SS1
```

---

### Часть 2: Настройка сервера хранения данных (LON-SS1)

На этом сервере мы развернем iSCSI-цель, которая будет предоставлять общий диск для кластера.

**2.1. Добавление диска для хранилища**

Добавьте к виртуальной машине LON-SS1 дополнительный виртуальный жесткий диск (например, 30 ГБ) для хранения данных кластера:

```powershell
# На физическом хосте Hyper-V
New-VHD -Path "C:\VHDs\LON-SS1-Data.vhdx" -SizeBytes 30GB -Dynamic
Add-VMHardDiskDrive -VMName "LON-SS1" -Path "C:\VHDs\LON-SS1-Data.vhdx"
```

Внутри LON-SS1 этот диск должен отображаться как "Неизвестный" и "Не инициализирован".

**2.2. Установка роли iSCSI Target Server**

Откройте **Server Manager** на LON-SS1 или используйте PowerShell:

```powershell
Install-WindowsFeature -Name FS-iSCSITarget-Server -IncludeManagementTools
```

**2.3. Создание iSCSI-виртуального диска и цели**

1.  В Server Manager перейдите в **File and Storage Services -> iSCSI**.
2.  Нажмите **"To create an iSCSI virtual disk, start the New iSCSI Virtual Disk Wizard"**.
3.  **Выберите диск:** Выберите добавленный вами дополнительный диск (например, `E:`).
4.  **Укажите имя:** Дайте имя виртуальному диску, например `ClusterStorage.vhdx`.
5.  **Укажите размер:** Задайте нужный размер (например, 20 ГБ).
6.  **Назначьте iSCSI Target:** Выберите **"New iSCSI target"** и дайте ему имя (например, `ClusterTarget`).
7.  **Укажите инициаторы:** Нажмите **Add** и добавьте IP-адреса узлов кластера:
    - `192.168.10.1` (LON-HOST1)
    - `192.168.10.2` (LON-HOST2)
8.  На шаге **Authentication** выберите **"Do not enable authentication"** (для упрощения лабораторной работы).
9.  Завершите создание цели.

**Альтернативный способ через PowerShell:**

```powershell
# Создаем iSCSI виртуальный диск
New-IscsiVirtualDisk -Path "E:\ClusterStorage.vhdx" -Size 20GB

# Создаем iSCSI цель
New-IscsiTarget -TargetName "ClusterTarget" -InitiatorIds @("IPAddress:192.168.10.1", "IPAddress:192.168.10.2")

# Привязываем диск к цели
Add-IscsiVirtualDiskTargetMapping -TargetName "ClusterTarget" -Path "E:\ClusterStorage.vhdx"
```

---

### Часть 3: Подготовка узлов кластера (LON-HOST1 и LON-HOST2)

Теперь нужно установить роли и компоненты для будущего кластера.

**3.1. Установка ролей**

На каждом узле (LON-HOST1 и LON-HOST2) выполните установку необходимых ролей:

```powershell
# Установка роли Hyper-V и компонентов для отказоустойчивого кластера
Install-WindowsFeature -Name Hyper-V, Failover-Clustering, RSAT-Clustering-PowerShell, Hyper-V-PowerShell, FS-FileServer -IncludeManagementTools

# Перезагрузка после установки
Restart-Computer
```

**3.2. Подключение к iSCSI-хранилищу**

Теперь оба узла должны увидеть диск, предоставленный LON-SS1.

**На LON-HOST1 и LON-HOST2:**
1.  Откройте **iSCSI Initiator** (можно найти через поиск или запустив `iscsicpl.exe`).
2.  На вкладке **Target** введите IP-адрес сервера LON-SS1 (`192.168.10.3`) и нажмите **Quick Connect**.
3.  Убедитесь, что статус "Connected". Нажмите **OK**.

**Альтернативный способ через PowerShell:**

```powershell
# На каждом узле кластера
# Запускаем службу iSCSI Initiator
Start-Service -Name MSiSCSI

# Подключаемся к цели
New-IscsiTargetPortal -TargetPortalAddress 192.168.10.3
Connect-IscsiTarget -NodeAddress (Get-IscsiTarget | Where-Object {$_.NodeAddress -like "*ClusterTarget*"}).NodeAddress -IsMultipathEnabled $true
```

**3.3. Инициализация и подготовка диска (ТОЛЬКО на LON-HOST1)**

Подготовьте общий диск к работе. **Важно:** Это делается только на одном узле кластера!

```powershell
# Получаем список дисков
Get-Disk | Where-Object {$_.OperationalStatus -eq "Offline"}

# Определяем номер нового диска (например, Disk 2) и инициализируем его
Initialize-Disk -Number 2 -PartitionStyle GPT

# Создаем том и форматируем его в NTFS
New-Partition -DiskNumber 2 -UseMaximumSize -DriveLetter S
Format-Volume -DriveLetter S -FileSystem NTFS -NewFileSystemLabel "ClusterStorage" -Confirm:$false
```

**Проверка на LON-HOST2:** Откройте **Дисковую оснастку (Disk Management)** на LON-HOST2 и убедитесь, что диск отображается как "Online" (он должен подхватиться автоматически).

---

### Часть 4: Создание отказоустойчивого кластера

**4.1. Проверка готовности кластера**

Перед созданием кластера выполните проверку на наличие ошибок:

```powershell
# Запускаем проверку (на любом узле)
Test-Cluster -Node LON-HOST1, LON-HOST2 -Include "Storage", "Network", "System Configuration"
```

Просмотрите отчет и убедитесь, что нет критических ошибок. Предупреждения можно игнорировать (например, о том, что нет контроллера домена для проверки DNS).

**4.2. Создание кластера**

Создайте кластер с именем `HyperVCluster`:

```powershell
New-Cluster -Name HyperVCluster -Node LON-HOST1, LON-HOST2 -StaticAddress 192.168.10.10
```

**4.3. Добавление общего хранилища в кластер**

Теперь нужно добавить общий iSCSI-диск в кластер:

1.  В **Failover Cluster Manager** разверните ваш кластер `HyperVCluster`.
2.  Перейдите в **Storage -> Disks**.
3.  Нажмите **Add Disk**.
4.  Выберите общий диск (он должен быть один) и нажмите **OK**.
5.  Диск должен появиться в списке со статусом "Online".

**Альтернативный способ через PowerShell:**

```powershell
# Добавляем все доступные диски в кластер
Add-ClusterDisk

# Получаем список дисков кластера
Get-ClusterDisk
```

**4.4. Настройка кворума**

Настройте кворум кластера для обеспечения отказоустойчивости:

1.  В **Failover Cluster Manager** перейдите в **Cluster Core Resources**.
2.  Выберите **More Actions -> Configure Cluster Quorum Settings**.
3.  Выберите **"Select the quorum witness"**.
4.  Выберите **"Configure a disk witness"** и укажите общий диск, который вы только что добавили.

**Альтернативный способ через PowerShell:**

```powershell
# Устанавливаем диск как свидетель кворума
Set-ClusterQuorum -DiskWitness "Cluster Disk 1"
```

---

### Часть 5: Настройка и проверка Hyper-V в кластере

**5.1. Настройка роли Hyper-V в кластере**

Теперь нужно превратить кластер в кластер Hyper-V. Это делается автоматически при создании виртуальных машин, но также можно настроить через PowerShell:

```powershell
# Включаем роль Hyper-V для кластера
Add-ClusterRole -Name "Hyper-V Role" -VMName "LON-HOST1"  # Здесь можно указать любую ВМ для инициализации
```

**5.2. Создание высокодоступной виртуальной машины**

Создайте тестовую виртуальную машину на общем хранилище, чтобы убедиться, что кластер работает:

```powershell
# Создаем ВМ на общем диске (путь S:\ClusterStorage\Volume01\)
New-VM -Name "TestVM" -Path "S:\ClusterStorage\Volume01\" -MemoryStartupBytes 2GB -SwitchName "Внешняя сеть" -Generation 2

# Делаем ВМ высокодоступной
Add-ClusterVirtualMachineRole -VMName "TestVM"

# Проверяем статус
Get-ClusterGroup
```

**5.3. Тестирование отказоустойчивости**

Проверьте перемещение виртуальной машины между узлами:

**Способ 1: Через Failover Cluster Manager**
1.  Выберите роль с вашей ВМ `TestVM`.
2.  Щелкните правой кнопкой мыши и выберите **Move -> Live Migration**.
3.  Выберите целевой узел (например, LON-HOST2).
4.  Наблюдайте за процессом миграции.

**Способ 2: Через PowerShell**
```powershell
# Перемещаем ВМ на другой узел
Move-ClusterVirtualMachineRole -Name "TestVM" -Node "LON-HOST2"

# Проверяем текущий владелец
Get-ClusterGroup -Name "TestVM" | Select-Object OwnerNode
```

**Способ 3: Симуляция сбоя**
Выключите один из узлов кластера (LON-HOST1 или LON-HOST2) и наблюдайте, как ВМ автоматически перезапускается на другом узле через некоторое время:

```powershell
# На физическом хосте выключите узел
Stop-VM -Name "LON-HOST1" -Force
```

---

### Часть 6: Дополнительные задания

**6.1. Настройка сети для Live Migration**

Для повышения производительности живой миграции настройте отдельную сеть:

1.  На каждом узле добавьте дополнительный сетевой адаптер (внутренняя сеть).
2.  Назначьте IP-адреса из отдельной подсети (например, 10.0.0.1 и 10.0.0.2).
3.  В **Failover Cluster Manager** перейдите в **Networks**.
4.  Выберите созданную сеть и укажите **"Allow this network for live migration"**.

**6.2. Настройка CSV (Cluster Shared Volumes)**

Общий диск, который вы добавили, уже является CSV. Вы можете проверить это:

```powershell
Get-ClusterSharedVolume
```

CSV позволяет узлам одновременно обращаться к одному и тому же LUN, что критически важно для Hyper-V.

**6.3. Создание нескольких виртуальных машин**

Создайте 2-3 дополнительные виртуальные машины на общем хранилище, используя разностные диски на основе базового образа (можно использовать тот же `base2025.vhdx`).

