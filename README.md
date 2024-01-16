<div>
<h1 align="left" style="display: flex;"> Massa Node tools </h1>
<img src="https://avatars.githubusercontent.com/u/92152619?s=200&v=4"  style="float: right;" width="100" height="100"></img>
</div>

Official documentation:
>- [Installing a node](https://docs.massa.net/docs/node/initial)


## Hardware Requirements
### Recommended Hardware Requirements 
Right now 4 cores and 8 GB of RAM should be enough to run a node, but it might increase in the future.

The balance_predictor script can be run on the same node without any significant additional system resources.

Author is not responsible for this script. All actions you perform are at your own peril and risk!

Script tested with Ubuntu 22.04

Update packages and Install dependencies

~~~bash
sudo apt update && sudo apt upgrade -y
sudo apt install pip -y
~~~

Check you python version. If >= "Python 3.10.12" - it's ok

~~~bash
python3 -V
~~~

set your password for massa-client

~~~bash
passwd=<YOUR_PASSWORD>
~~~

Copy and paste this code below in you terminal

~~~bash
tee ~/balance_predictor.py > /dev/null <<EOF
import time
import logging
from datetime import datetime, timedelta
import subprocess
import re


# Путь к клиенту Massa и параметры конфигурации
massa_client_path = "~/massa/massa-client/"
massa_client_executable = "./massa-client"
password = "$passwd"  # Пароль для доступа к клиенту Massa
transaction_fee = "0"  # Комиссия за транзакцию


# Функция для выполнения команд в Massa Client
def run_massa_command(command):
    try:
        process = subprocess.Popen(
            f"cd {massa_client_path} && echo {password} | {massa_client_executable} {command}",
            shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True
        )
        stdout, stderr = process.communicate(input=password + "\n")
        if stderr:
            raise Exception(stderr)
        return stdout
    except Exception as e:
        logging.error(f"Ошибка при выполнении команды: {e}")
        print(f"Ошибка при выполнении команды: {e}")
        raise

# Функция для извлечения информации о кошельке
def get_wallet_info():
    wallet_info_output = run_massa_command("-p $passwd wallet_info")
    address_match = re.search(r"Address: (\w+) \(thread \d+\):", wallet_info_output)
    balance_match = re.search(r"Balance: final=(\d+.\d+),", wallet_info_output)
    rolls_match = re.search(r"Rolls: active=(\d+), final=(\d+), candidate=(\d+)", wallet_info_output)

    if address_match and balance_match and rolls_match:
        wallet_address = address_match.group(1)
        balance = float(balance_match.group(1))
        active_rolls = int(rolls_match.group(1))
        final_rolls = int(rolls_match.group(2))
        candidate_rolls = int(rolls_match.group(3))

        logging.info(f"Адрес кошелька: {wallet_address}, Баланс: {balance}, Роллы: активные={active_rolls}, финальные={final_rolls}, кандидатские={candidate_rolls}")
#        print(f"Адрес кошелька: {wallet_address}, Баланс: {balance}, Роллы: активные={active_rolls}, финальные={final_rolls}, кандидатские={candidate_rolls}")
        return wallet_address, balance, active_rolls, final_rolls, candidate_rolls
    else:
        logging.warning("Не удалось получить информацию о кошельке.")
        print("Не удалось получить информацию о кошельке.")
        return None, None, None, None, None


### Настройка логирования
logging.basicConfig(filename='balance_predictor.log', level=logging.INFO,
                    format='%(asctime)s %(levelname)s:%(message)s')

# Функция get_wallet_info должна быть здесь или импортирована

# Функция для преобразования секунд в дни, часы, минуты, секунды
def format_time(seconds):
    days, seconds = divmod(seconds, 86400)
    hours, seconds = divmod(seconds, 3600)
    minutes, seconds = divmod(seconds, 60)
    return f"{int(days)}д {int(hours)}ч {int(minutes)}м {int(seconds)}с"

# Функция для анализа и прогнозирования баланса
def analyze_and_predict_balance():
    balance_history = []
    last_logged_time = datetime.now()
    
    while True:
        _, current_balance, _, _, _ = get_wallet_info()
        
        if current_balance is not None:
            balance_history.append((datetime.now(), current_balance))
            
            if len(balance_history) > 100:
                balance_history.pop(0)
                
            if len(balance_history) == 100:
                # Расчет средней скорости изменения баланса
                time_diff = (balance_history[-1][0] - balance_history[0][0]).total_seconds()
                balance_diff = balance_history[-1][1] - balance_history[0][1]
                balance_rate = balance_diff / time_diff
                
                # Предсказание времени достижения баланса 100
                if balance_rate != 0:
                    time_to_reach = (100 - current_balance) / balance_rate
                    estimated_time = datetime.now() + timedelta(seconds=time_to_reach)
                    formatted_time = format_time(time_to_reach)
                    message = f"Текущий баланс: {current_balance}, Предполагаемое время достижения баланса 100: {estimated_time.strftime('%Y-%m-%d %H:%M:%S')} ({formatted_time})"
                    
                    # Логирование и вывод в консоль раз в 5 минут
                    if (datetime.now() - last_logged_time).total_seconds() >= 300:
                        print(message)
                        logging.info(message)
                        last_logged_time = datetime.now()
                else:
                    message = "Баланс не изменяется."
                    if (datetime.now() - last_logged_time).total_seconds() >= 300:
                        print(message)
                        logging.info(message)
                        last_logged_time = datetime.now()
        
        time.sleep(10)

# Запуск функции анализа и прогнозирования
try:
    analyze_and_predict_balance()
except Exception as e:
    logging.error(f"Ошибка во время анализа и прогнозирования баланса: {e}")
    print(f"Ошибка во время анализа и прогнозирования баланса: {e}")

EOF
~~~

Change exec options for balance_predictor.py

~~~bash
chmod +x ~/balance_predictor.py
~~~


Start script in you terminal

~~~bash
python3 balance_predictor.py
~~~


