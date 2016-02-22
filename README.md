## Удобный способ организовать примитивные параллельный вычисления в Jupyter (IPython Notebook)

Иногда возникает желание запустить вычисления в ячейке и, не дожидаясь пока они закончатся, продолжить работу. Например, нужно скачать 1000 урлов и достать у них заголовки страниц. Хорошо бы запустить процесс скачивания и сразу начать отлаживать код для выделения заголовков.

Можно использовать такой код:
```python
def jobs_manager():
    from IPython.lib.backgroundjobs import BackgroundJobManager
    from IPython.core.magic import register_line_magic
    from IPython import get_ipython
    
    jobs = BackgroundJobManager()

    @register_line_magic
    def job(line):
        ip = get_ipython()
        jobs.new(line, ip.user_global_ns)

    return jobs
```

Использовать его можно так:

<img src="https://habrastorage.org/files/9ed/bf0/1d6/9edbf01d60a246159b551fb50d0a32e7.gif"/>

`jobs` ведёт учёт фоновых операций:

<img src="https://habrastorage.org/files/16b/b9d/fe5/16bb9dfe532f4442ae6c10958607b87b.gif"/>

Чтобы убить операцию нужно использовать специальный хак:
```python
def kill_thread(thread):
    import ctypes
    
    id = thread.ident
    code = ctypes.pythonapi.PyThreadState_SetAsyncExc(
        ctypes.c_long(id),
        ctypes.py_object(SystemError)
    )
    if code == 0:
        raise ValueError('invalid thread id')
    elif code != 1:
        ctypes.pythonapi.PyThreadState_SetAsyncExc(
            ctypes.c_long(id),
            ctypes.c_long(0)
        )
        raise SystemError('PyThreadState_SetAsyncExc failed')
```

Использовать так:

<img src="https://habrastorage.org/files/b08/c1b/9ae/b08c1b9aef084f52bfbcb671dc84c34f.gif"/>

`jobs` также копит стектрейсы (`kill_thread` кидает `SystemError` внутри треда):

<img src="https://habrastorage.org/files/c16/c00/efc/c16c00efc4ab4622acb03c43372e8774.gif"/>

`%job` можно даже использовать как альтернативу `multiprocessing.ThreadPool`:

<img src="https://habrastorage.org/files/75f/dbd/93b/75fdbd93bdf64e3c99ecfbe0bf9181bb.gif"/>

Нарезать последовательность на заданное количество кусочков удобно функцией:
```python
def get_chunks(sequence, count):
    count = min(count, len(sequence))
    chunks = [[] for _ in range(count)]
    for index, item in enumerate(sequence):
        chunks[index % count].append(item) 
    return chunks
```

Завершить работу пула можно так:

<img src="https://habrastorage.org/files/e05/656/472/e056564721ca45c9bf5b9033fc0e2f91.gif"/>

### Что нужно понимать

  1. `%job` работает на тредах, поэтому нужно помнить про GIL. `%job` — это не про распараллеливание тяжелых вычислений, которые происходят в питоновом байт-коде. Это про IO-bound операции и вызов внешних утилит.

  2. Нельзя нормально завершить произвольный код внутри треда, поэтому в `kill_thread` используется хак. Этот хак работает не всегда. Например, если код внутри треда выполняет `sleep` исключение, которое кидает `kill_thread` игнорируется.

  3. Код в `%job` выполняется через `eval`. Грубо говоря, можно использовать выражения, который могут встречаться после знака `=`. Никаких `print` и присваиваний. Впрочем, всегда можно завернуть сложный код в функцию и выполнить `%job f()`

  4. Передача сообщений из %job выполняется через жёсткий диск. Например, нельзя непосредственно получить содержание скачанной страницы, нужно его сохранить на диск, а потом прочитать.
