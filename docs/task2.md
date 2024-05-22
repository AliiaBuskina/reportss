### Задача 2. Параллельный парсинг веб-страниц с сохранением в базу данных
Задача: Напишите программу на Python для параллельного парсинга нескольких веб-страниц с сохранением данных в базу данных с использованием подходов threading, multiprocessing и async. Каждая программа должна парсить информацию с нескольких веб-сайтов, сохранять их в базу данных.

Подробности задания:

Напишите три различных программы на Python, использующие каждый из подходов: threading, multiprocessing и async.
Каждая программа должна содержать функцию parse_and_save(url), которая будет загружать HTML-страницу по указанному URL, парсить ее, сохранять заголовок страницы в базу данных и выводить результат на экран.
Используйте базу данных из лабораторной работы номер 1 для заполенния ее данными. Если Вы не понимаете, какие таблицы и откуда Вы могли бы заполнить с помощью парсинга, напишите преподавателю в общем чате потока.
Для threading используйте модуль threading, для multiprocessing - модуль multiprocessing, а для async - ключевые слова async/await и модуль aiohttp для асинхронных запросов.
Создайте список нескольких URL-адресов веб-страниц для парсинга и разделите его на равные части для параллельного парсинга.
Запустите параллельный парсинг для каждой программы и сохраните данные в базу данных.
Замерьте время выполнения каждой программы и сравните результаты.


#### threading
```python
import threading
import requests
import psycopg2
from time import time

def save_to_db(data):
    conn = psycopg2.connect("dbname=openalex_db user=postgres password=Aliya2103 host=localhost")
    cur = conn.cursor()
    for record in data:
        cur.execute(
            "INSERT INTO openalex_data (id, title, abstract, domain) VALUES (%s, %s, %s, %s)",
            (record['id'], record['title'], record['abstract'], record['domain'])
        )
    conn.commit()
    cur.close()
    conn.close()

def dict_to_sentence(dictionary):
    sentence_words = [''] * (max(max(indices) for indices in dictionary.values()) + 1)
    for word, indices in dictionary.items():
        for index in indices:
            sentence_words[index] = word
    return ' '.join(sentence_words)

def parse_and_save(url):
    response = requests.get(url)
    page_with_results = response.json()
    all_data = []
    if 'results' in page_with_results:
        for result in page_with_results['results']:
            primary_topic = result.get('primary_topic')
            if primary_topic:
                domain_display_name = primary_topic.get('domain', {}).get('display_name', '')
            else:
                domain_display_name = ''
            if domain_display_name:
                title = result.get('title')
                if title:
                    record = {'id': result['id'].replace("https://openalex.org/", ""), 'domain': domain_display_name}
                    for key, value in result.items():
                        if key != 'id':
                            record[key] = value
                    if 'abstract_inverted_index' in record and record['abstract_inverted_index']:
                        record['abstract'] = dict_to_sentence(record['abstract_inverted_index'])
                        all_data.append(record)
    save_to_db(all_data)

def main_threading():
    start_time = time()
    urls = [f'https://api.openalex.org/works?page={i}&per-page=200' for i in range(1, 7)]
    threads = []

    for url in urls:
        thread = threading.Thread(target=parse_and_save, args=(url,))
        thread.start()
        threads.append(thread)

    for thread in threads:
        thread.join()

    print(f"Time taken (threading): {time() - start_time} seconds")

if __name__ == "__main__":
    main_threading()
```
![threading](threading.png)

#### multiprocessing
```python
import multiprocessing
import requests
import psycopg2
from time import time


def save_to_db(data):
    conn = psycopg2.connect("dbname=openalex_db user=postgres password=Aliya2103 host=localhost")
    cur = conn.cursor()
    for record in data:
        cur.execute(
            "INSERT INTO openalex_data (id, title, abstract, domain) VALUES (%s, %s, %s, %s)",
            (record['id'], record['title'], record['abstract'], record['domain'])
        )
    conn.commit()
    cur.close()
    conn.close()

def dict_to_sentence(dictionary):
    sentence_words = [''] * (max(max(indices) for indices in dictionary.values()) + 1)
    for word, indices in dictionary.items():
        for index in indices:
            sentence_words[index] = word
    return ' '.join(sentence_words)

def parse_and_save(url):
    response = requests.get(url)
    page_with_results = response.json()
    all_data = []
    if 'results' in page_with_results:
        for result in page_with_results['results']:
            primary_topic = result.get('primary_topic')
            if primary_topic:
                domain_display_name = primary_topic.get('domain', {}).get('display_name', '')
            else:
                domain_display_name = ''
            if domain_display_name:
                title = result.get('title')
                if title:
                    record = {'id': result['id'].replace("https://openalex.org/", ""), 'domain': domain_display_name}
                    for key, value in result.items():
                        if key != 'id':
                            record[key] = value
                    if 'abstract_inverted_index' in record and record['abstract_inverted_index']:
                        record['abstract'] = dict_to_sentence(record['abstract_inverted_index'])
                        all_data.append(record)
    save_to_db(all_data)

def main_multiprocessing():
    start_time = time()
    urls = [f'https://api.openalex.org/works?page={i}&per-page=200' for i in range(7, 13)]
    processes = []

    for url in urls:
        process = multiprocessing.Process(target=parse_and_save, args=(url,))
        process.start()
        processes.append(process)

    for process in processes:
        process.join()

    print(f"Time taken (multiprocessing): {time() - start_time} seconds")

if __name__ == "__main__":
    main_multiprocessing()
```
![multiprocessing](multiprocessing.png)

#### async
```python
import aiohttp
import asyncio
import asyncpg
import psycopg2
from time import time

def create_aiohttp_session():
    return aiohttp.ClientSession()


async def save_to_db(data):
    conn = await asyncpg.connect('postgresql://postgres:Aliya2103@localhost:5432/openalex_db')
    try:
        for record in data:
            await conn.execute(
                "INSERT INTO openalex_data (id, title, abstract, domain) VALUES ($1, $2, $3, $4)",
                record['id'], record['title'], record['abstract'], record['domain']
            )
    finally:
        await conn.close()

async def dict_to_sentence(dictionary):
    sentence_words = [''] * (max(max(indices) for indices in dictionary.values()) + 1)
    for word, indices in dictionary.items():
        for index in indices:
            sentence_words[index] = word
    return ' '.join(sentence_words)

async def parse_and_save(url):
    async with aiohttp.ClientSession(connector=aiohttp.TCPConnector(ssl=False)) as session:
        async with session.get(url) as response:
            page_with_results = await response.json()
            all_data = []
            if 'results' in page_with_results:
                for result in page_with_results['results']:
                    primary_topic = result.get('primary_topic')
                    if primary_topic:
                        domain_display_name = primary_topic.get('domain', {}).get('display_name', '')
                    else:
                        domain_display_name = ''
                    if domain_display_name:
                        title = result.get('title')
                        if title:
                            record = {'id': result['id'].replace("https://openalex.org/", ""),
                                      'domain': domain_display_name}
                            for key, value in result.items():
                                if key != 'id':
                                    record[key] = value
                            if 'abstract_inverted_index' in record and record['abstract_inverted_index']:
                                record['abstract'] = await dict_to_sentence(record['abstract_inverted_index'])
                                all_data.append(record)
            await save_to_db(all_data)

async def main_async():
    start_time = time()
    urls = [f'https://api.openalex.org/works?page={i}&per-page=200' for i in range(20, 26)]
    tasks = [parse_and_save(url) for url in urls]
    await asyncio.gather(*tasks)
    print(f"Time taken (async): {time() - start_time} seconds")

if __name__ == "__main__":
    asyncio.run(main_async())
```
![asyncio](async.png)

* Асинхронное программирование, применяемое в подходе с async, обеспечивает значительное преимущество по времени выполнения по сравнению с использованием multiprocessing и threading. Этот метод позволяет выполнять несколько задач одновременно без блокировки основного потока, что особенно полезно для операций ввода-вывода, таких как парсинг веб-страниц.