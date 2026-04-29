import requests
import uuid
import json
import time

# ========== API-КЛЮЧ ==========
# Сюда вставь свою авторизационную строку из личного кабинета 
AUTH_KEY = "GigaCHAT API-Key"
# ===================================

# Функция для получения токена доступа (действует 30 минут)
def get_access_token():
    url = "https://ngw.devices.sberbank.ru:9443/api/v2/oauth"
    headers = {
        'Authorization': f'Basic {AUTH_KEY}',
        'RqUID': str(uuid.uuid4()),
        'Content-Type': 'application/x-www-form-urlencoded'
    }
    payload = 'scope=GIGACHAT_API_PERS'
    
    try:
        # Игнорируем проверку SSL-сертификата, как указано в инструкциях на GitHub [citation:2]
        response = requests.post(url, headers=headers, data=payload, verify=False)
        response.raise_for_status() # Проверяем, не вернул ли сервер ошибку
        return response.json()['access_token']
    except requests.exceptions.RequestException as e:
        print(f"❌ Ошибка получения токена: {e}")
        if response.status_code == 401:
            print("   Неверный API-ключ. Проверь AUTH_KEY.")
        exit()

# Основная функция генерации
def generate_kp(company, industry, budget, timeline, problem):
    print("🔑 Получаю токен доступа...")
    token = get_access_token()
    
    print("🤖 Отправляю запрос к GigaChat...")
    url = "https://gigachat.devices.sberbank.ru/api/v1/chat/completions"
    headers = {
        'Authorization': f'Bearer {token}',
        'Content-Type': 'application/json',
        'Accept': 'application/json'
    }
    
    system_prompt = """Ты — эксперт по B2B-продажам IT-решений. Твоя задача — создавать коммерческие предложения (КП) по шаблону.
📋 Шаблон КП должен содержать:
a. Заголовок {Коммерческое предложение для [Название компании]};
b. Цели и задачи проекта;
c. Project-scope (этапы проекта);
d. Техническая реализация решения;
e. Результаты внедрения (с цифрами);
f. Бюджет и сроки;
g. Следующие шаги;
h. Допущения и ограничения.

📜 Правила: Пиши на русском, деловым, но не сухим языком. Не используй шаблонные фразы. Добавь конкретику: цифры, сроки, примеры."""

    user_prompt = f"""
Компания: {company}
Отрасль: {industry}
Бюджет: {budget} руб.
Сроки: {timeline} мес.
Потребность: {problem}

Создай коммерческое предложение по шаблону.
"""
    payload = {
        "model": "GigaChat", # или GigaChat-Pro
        "messages": [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt}
        ],
        "temperature": 0.7,
        "max_tokens": 2500
    }
    
    try:
        response = requests.post(url, headers=headers, json=payload, verify=False)
        response.raise_for_status()
        result = response.json()
        return result['choices'][0]['message']['content']
    except requests.exceptions.RequestException as e:
        print(f"❌ Ошибка генерации: {e}")
        if response.status_code == 402:
            print("   Не хватает токенов на балансе. Пополни счет в личном кабинете.")
        elif response.status_code == 401:
            print("   Недействительный токен доступа. Возможно, истекло время.")
        exit()

# Запуск программы
if __name__ == "__main__":
    print("\n🤖 Генератор КП GigaChat (консольная версия)")
    print("-" * 40)
    company = input("Название компании: ")
    industry = input("Отрасль: ")
    budget = input("Бюджет (₽): ")
    timeline = input("Срок (мес): ")
    problem = input("Потребность/проблема: ")
    
    print("\n⏳ Генерация коммерческого предложения...\n")
    kp = generate_kp(company, industry, budget, timeline, problem)
    
    print("\n" + "="*60)
    print("📄 ВАШЕ КОММЕРЧЕСКОЕ ПРЕДЛОЖЕНИЕ")
    print("="*60)
    print(kp)
    print("="*60)
    input("\nНажми Enter, чтобы выйти...")
