### Плохой CI/CD файл

```yaml
name: Bad CI/CD Workflow

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      # Нахардкодим секретов
      - name: Set up environment variables
        run: |
          echo "DB_PASSWORD=my_pswd" >> $GITHUB_ENV
          echo "SSH_PASSWORD=jdfslakfjl;dsak" >> $GITHUB_ENV

      - name: Install dependencies
        run: npm install

	 # Некое олициетворение команды которая не несет пользы и засоряет логи
     - name: Print working directory
      run: pwd

      # Нафиг кэш
      - name: Build project
        run: npm run build

      # Деплоим прям на сервер (без обработки ошибок)
      - name: Deploy to production
        if: github.ref == 'refs/heads/main'
        run: sshpass -p 'lolkekcheburek' ssh mrlargha@mrlargha.ru "cd /var/www/okno && git pull origin main && npm install && systemctl restart nginx"
```

### Хороший CI/CD файл

```yaml
name: Good CI/CD Workflow

on:
  push:
    branches: [ main ]

env:
  DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
  API_KEY: ${{ secrets.API_KEY }}

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Cache node modules
        id: cache-node-modules
        uses: actions/cache@v3
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-

      - name: Install dependencies
        if: steps.cache-node-modules.outputs.cache-hit != 'true'
        run: npm ci

      # Кэш загрузили
      - name: Build project
        run: npm run build

      # Теперь у нас все делается через SSH key и с обработкой ошибок
      - name: Deploy to production
        if: github.ref == 'refs/heads/main'
        env:
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
        run: |
          mkdir -p ~/.ssh/
          echo "${SSH_PRIVATE_KEY}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan -H mrlargha.ru >> ~/.ssh/known_hosts
          set +e
          result=$(ssh mrlargha@mrlargha.ru "cd /var/www/okno && git pull origin main && npm install && systemctl reload nginx")
          exit_code=$?
          set -e
          if [[ "$exit_code" -ne 0 ]]; then
            echo "Deployment failed: $result"
            exit 1
          else
            echo "Successfully deployed!"
          fi
```

### Описание того что мы наворотили

#### 1. **Хардкодинг секретов в рабочем процессе**
Чувствительные данные: в моем примере пароль от бд и рандомный ключ API хранятся прямо в коде, что потенциально может привести к утечке этих данных. (Если сейчас поиском пройтись по гитхабу то можно найти ключи от любого нужного вам сервиса, гитхаб с этим активно борется, но лайфхаком можно пользоваться)
#### 2. **Отсутствие кеширования зависимостей**

При каждом запуске процесса происходит установка зависимостей npm что очень сильно тормозит сборку, зависимости можно закешировать и переиспользовать:
    
    ```yaml
    - name: Cache node modules
      id: cache-node-modules
      uses: actions/cache@v3
      with:
        path: ~/.npm
        key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
        restore-keys: |
          ${{ runner.os }}-node-
    ```
Используем хэш отт package-lock.json как ключ для хранения нашех кэшей
#### 3. **Ненужный шаг сборки**
Имеется непонятный и ненужный при сборки шаг, иногда так делают для отладки но из итогового процесса такое лучше убирать чтобы не перегружать сборку
    
    ```yaml
    - name: Print working directory
      run: pwd
    ```
    

#### 4. **Недостаток обработки ошибок**

Наш шаг развертывания не обрабатывает возможные ошибки. Мы не сможем понять что наше чудо незадеплоилось, а это очень печально, не надо так
    ```yaml
    set +e
    result=$(ssh user@production-server.com "...")
    exit_code=$?
    set -e
    if [[ "$exit_code" -ne 0 ]]; then
      echo "Deployment failed: $result"
      exit 1
    else
      echo "Successfully deployed!"
    fi
    ```
#### 5. **Использование хардкода для подключения по SSH**
Плохая практика хардкодинг пароля от продакшн сервера (ужас какой) и использование sshpass (зачем его придумали вообще), самое лучшее решение это использование ключей-секретов, как я делаю для своего сервера и как сделано у нас на работе
