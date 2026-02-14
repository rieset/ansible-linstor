# Combined узлы в LINSTOR

## Что такое Combined узел?

**Combined узел** = Controller + Satellite в одном узле

- Выполняет функции контроллера (управление кластером)
- Выполняет функции satellite (хранение данных)
- Экономит ресурсы (не нужны отдельные узлы для controller и satellite)

## Как создать Combined узел?

**Просто добавьте узел в ОБЕ группы одновременно:**

```ini
[controller]
192.168.1.11
192.168.1.12
192.168.1.13

[satellite]
192.168.1.11  # тот же узел
192.168.1.12  # тот же узел
192.168.1.13  # тот же узел
```

## Логика определения типа узла

Playbook автоматически определяет тип узла:

### 1. Combined (если узел в ОБЕИХ группах)
```yaml
# roles/linstor/controller/tasks/main.yaml:60-64
- name: initialize the LINSTOR control node as a Combined type
  shell: linstor node create ... --node-type Combined
  when: "'satellite' in group_names"  # если узел в группе satellite
```

### 2. Controller (если узел ТОЛЬКО в группе controller)
```yaml
# roles/linstor/controller/tasks/main.yaml:54-58
- name: initialize the LINSTOR control node as pure Controller
  shell: linstor node create ... --node-type Controller
  when: "'satellite' not in group_names"  # если узел НЕ в группе satellite
```

### 3. Satellite (если узел ТОЛЬКО в группе satellite)
```yaml
# roles/linstor/satellite/tasks/main.yaml:63-66
- name: join LINSTOR cluster as satellite node
  shell: linstor node create ... --node-type Satellite
```

## Текущая конфигурация

**Ваша текущая конфигурация УЖЕ настроена как Combined:**

```ini
[controller]
192.168.1.11
192.168.1.12
192.168.1.13

[satellite]
192.168.1.11  # ✅ Combined
192.168.1.12  # ✅ Combined
192.168.1.13  # ✅ Combined
```

**Результат:** Все 3 узла будут созданы как **Combined** узлы.

## Преимущества Combined узлов

### ✅ Плюсы:
1. **Экономия ресурсов** - не нужны отдельные узлы для controller
2. **Простота** - меньше узлов для управления
3. **Подходит для небольших кластеров** - 3-5 узлов
4. **Высокая доступность** - 3 Combined узла обеспечивают HA для controller и storage

### ⚠️ Минусы:
1. **Больше нагрузка на узел** - выполняет две роли
2. **Меньше изоляции** - проблемы с storage могут повлиять на controller
3. **Меньше масштабируемости** - сложнее масштабировать controller и storage независимо

## Рекомендации

### Когда использовать Combined:
- ✅ Небольшие кластеры (3-7 узлов)
- ✅ Тестовые/разработческие окружения
- ✅ Ограниченные ресурсы
- ✅ Простые deployment'ы

### Когда НЕ использовать Combined:
- ❌ Большие production кластеры (10+ узлов)
- ❌ Критичные production окружения
- ❌ Нужна независимая масштабируемость controller и storage
- ❌ Строгие требования к изоляции

## Архитектура с Combined узлами

```
┌─────────────────────────────────────────┐
│      LINSTOR Cluster (Combined)          │
├─────────────────────────────────────────┤
│                                          │
│  Combined Nodes (Controller + Satellite) │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐
│  │ .1.11    │  │ .1.12    │  │ .1.13    │
│  │ Combined │  │ Combined │  │ Combined │
│  │          │  │          │  │          │
│  │ Controller│  │ Controller│  │ Controller│
│  │ Satellite │  │ Satellite │  │ Satellite │
│  │ Storage   │  │ Storage   │  │ Storage   │
│  └──────────┘  └──────────┘  └──────────┘
│       │              │              │
│       └──────────────┴──────────────┘
│         DRBD Replication + HA
│                                          │
└─────────────────────────────────────────┘
```

## Storage Pools для Combined узлов

### Вариант 1: Без storage pool (file-thin только)
```ini
[satellite]
192.168.1.11
192.168.1.12
192.168.1.13

# НЕТ группы linstor_storage_pool
```
→ Создастся только file-thin pool на каждом узле

### Вариант 2: С storage pool (lvm-thin)
```ini
[satellite]
192.168.1.11
192.168.1.12
192.168.1.13

[linstor_storage_pool]
192.168.1.11  # если есть диск /dev/sdb
192.168.1.12  # если есть диск /dev/sdb
192.168.1.13  # если есть диск /dev/sdb
```
→ Создастся lvm-thin pool на узлах с дисками

**Важно:** Убедитесь, что на узлах есть свободный диск `/dev/sdb` (или другой, указанный в `group_vars/all.yaml`)

## Проверка после установки

После выполнения playbook проверьте тип узлов:

```bash
# На любом узле
linstor node list
```

Должны увидеть:
```
┌─────────────────────────────────────────┐
│ Node      │ Address    │ Type    │ ... │
├───────────┼────────────┼─────────┼─────┤
│ node-11    │ 192.168.1.11 │ Combined│ ... │
│ node-12    │ 192.168.1.12 │ Combined│ ... │
│ node-13    │ 192.168.1.13 │ Combined│ ... │
└───────────┴────────────┴─────────┴─────┘
```

## Итог

**Ваша текущая конфигурация уже оптимальна для Combined узлов!**

Все 3 узла будут работать как Combined, что идеально для кластера из 3 узлов:
- ✅ Высокая доступность (3 контроллера)
- ✅ Репликация данных (3 satellite)
- ✅ Экономия ресурсов
- ✅ Простота управления

