# `Deploy action` — Деплой приложения в Kubernetes

**Автор:** `demid1984`
**Тип действия:** Composite
**Назначение:** Запуск деплоя приложения в Kubernetes-кластер.

---

## Описание

Действие автоматизирует процесс развёртывания Java-приложения (собранного через Maven) в Kubernetes-кластере.
Действие:
- Получает версию и имя приложения из `pom.xml` через Maven;
- Подставляет переменные окружения и параметры в шаблоны манифестов `deployment.yaml` и `service.yaml`;
- Обновляет `ConfigMap` и (при наличии) расшифрованный `Secret`;
- Применяет обновлённые манифесты и ожидает успешное завершение `rollout`.

---

## Обязательные входные параметры (`inputs`)

| Параметр | Описание | Тип | Обязательный |
|---------|----------|-----|--------------|
| `deployment-file-path` | Путь к шаблону файла `deployment.yaml` (например, `kubernetes/deployment.yaml`) | `string` | ✅ Да |
| `service-file-path` | Путь к шаблону файла `service.yaml` (например, `kubernetes/service.yaml`) | `string` | ✅ Да |
| `kube-config` | Base64-кодированный или обычный `kubeconfig` для подключения к кластеру | `string` | ✅ Да |
| `kube-namespace` | Kubernetes-пространство имён для развертывания | `string` | ✅ Да |

---

## Необязательные входные параметры (`inputs`)

| Параметр | Описание | Значение по умолчанию | Тип |
|---------|----------|----------------------|-----|
| `gpg-passphase` | Пароль для расшифровки GPG-архива секретов (`secrets.properties.gpg`). Если не задан — пропускается этап обновления секретов. | `''` | `string` |
| `deploy-timeout` | Таймаут ожидания завершения развёртывания (в секундах). | `'300'` | `string` |
| `env-vars-multiline` | Переменные окружения для подстановки в манифесты (формат: `KEY=VALUE`, по одной на строку). Поддерживаются комментарии через `#`. | — | `string` (multiline) |

> ⚠️ **Важно:** Все переменные окружения (включая `APPLICATION_NAME` и `VERSION`, выведенные через Maven) подставляются в шаблоны через `envsubst`.

---

## Ожидаемая структура репозитория

Для корректной работы действия требуется следующая структура директорий:

```
├── kubernetes/
│   ├── deployment.yaml        # ← шаблон (поддерживает переменные вида ${VAR_NAME})
│   ├── service.yaml           # ← шаблон
│   ├── config/
│   │   └── <namespace>/
│   │       └── application.properties  # конфигурация → создаётся ConfigMap
│   └── secrets/
│       └── <namespace>/
│           ├── secrets.properties.gpg    # зашифрованные секреты
│           └── (опционально) secrets.properties  # расшифровывается во время выполнения
```

Где `<namespace>` — значение параметра `kube-namespace`.

---

## Как работает действие (по шагам)

1. **Checkout**
   Клонирует репозиторий с текущей веткой (`github.ref`), с полной историей (`fetch-depth: 0`).

2. **Извлечение метаданных проекта**
   С помощью Maven извлекаются:
    - `project.artifactId` → `APPLICATION_NAME`
    - `project.version` → `VERSION`
      Эти значения сохраняются в `GITHUB_OUTPUT` и позже загружаются в `GITHUB_ENV`.

3. **Настройка переменных окружения**
    - Пользовательские переменные из `env-vars-multiline` парсятся и сохраняются в `GITHUB_ENV`.
    - Добавляются системные переменные: `APPLICATION_NAME` и `VERSION`.

4. **Генерация манифестов Kubernetes**
   `envsubst` подставляет переменные окружения в шаблоны:
   ```bash
   envsubst < kubernetes/deployment.yaml > deployment.yaml
   envsubst < kubernetes/service.yaml > service.yaml
   ```

5. **Аутентификация в Kubernetes**
   Используется действие [`azure/k8s-set-context@v3`](https://github.com/azure/k8s-set-context) для установки контекста кластера.

6. **Обновление ConfigMap**
   Удаляется старый ConfigMap (`<app-name>-config-<namespace>`) и создаётся новый из `kubernetes/config/<namespace>/application.properties`.

7. **Обновление Secret (опционально)**
   Если указан `gpg-passphase`:
    - Расшифровывается `secrets.properties.gpg` в `secrets.properties`.
    - Удаляется старый Secret (`<app-name>-<namespace>`).
    - Создаётся новый Secret из `secrets.properties`.
    - Расшифрованный файл удаляется (для безопасности).

8. **Применение манифестов**
   Выполняются команды:
   ```bash
   kubectl apply -f deployment.yaml
   kubectl apply -f service.yaml
   ```

9. **Ожидание завершения развёртывания**
   ```bash
   kubectl rollout status deployment/<app-name> --timeout=<deploy-timeout>s
   ```
   Действие завершится ошибкой, если за указанное время развёртывание не завершится успешно.

---

## Пример использования в workflow

```yaml
- name: Deploy to Kubernetes
  uses: demid1984/maven-deploy-action@v0.0.3
  with:
    deployment-file-path: kubernetes/deployment.yaml
    service-file-path: kubernetes/service.yaml
    kube-config: ${{ secrets.KUBE_CONFIG }}
    kube-namespace: production
    deploy-timeout: 600
    env-vars-multiline: |
      # Описание
      APP_TITLE=MySuperApp
      LOG_LEVEL=INFO
    gpg-passphase: ${{ secrets.GPG_PASSPHRASE }}
```

> 💡 **Совет:** Значения `kube-config` и `gpg-passphase` рекомендуется хранить в `secrets` GitHub.

---

## Требования

- Работает в **GitHub-hosted runner** (Ubuntu).
- В проекте есть maven-wrapper mvnw (если нет, создается командой mvn wrapper:wrapper)
- Bash (`envsubst`, `gpg`, `kubectl`) доступен в окружении.
- Структура файлов и именований соответствует ожидаемой (см. выше).

---

## Известные особенности

- В параметре `gpg-passphase` **обязательно** указывать пароль без переносов строки — `--passphrase-fd 0` требует строго одного потока.
- Имена `ConfigMap` и `Secret` формируются по шаблону:
  `<application-name>-config-<namespace>` и `<application-name>-<namespace>` соответственно.
- Пустые строки и строки, начинающиеся с `#` в `env-vars-multiline`, игнорируются.

---

## 🤝 Вклад в проект

Приветствуются PR и issues!
Следуйте стандартам: проверяйте форматирование, добавляйте тесты, описывайте изменения.

---

© 2026, demid1984
Сделано с ❤️ для надёжных CI/CD потоков.
