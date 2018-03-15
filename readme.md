

```python
from pathlib import Path
import typing as T
import json
import os


from eliot import start_action, to_file, add_destinations
from tinydb import TinyDB
import requests


with open('db.json', 'w'):
        pass


to_file(open('log', 'w'))


def to_json_file():
    json_file = Path('log.json')
    json_file.write_text(json.dumps([]))
    
    def write_message(message: dict):
        messages = json.loads(json_file.read_text())
        messages.append(message)
        json_file.write_text(json.dumps(messages))
    
    return write_message


add_destinations(to_json_file())
```

## Contrived example of [eliot](http://eliot.readthedocs.io/en/0.9.0/)

This is just meant to show how we might go about using eliot to log the following steps:

1. download the contents of a url
2. marshal the content into json
3. persist the marshaled content to a database


```python
class MarshaledResponse(T.NamedTuple):
    """The result of a url being marshaled into json."""
    string: str     # the json-encoded string
    from_url: str   # the url the string was marshaled from 


def download(url: str) -> requests.Response:
    """Download the url and return a response."""
    with start_action(action_type='download', url=url):
        return requests.get(url)


def marshal(response: requests.Response) -> T.Optional[MarshaledResponse]:
    """Return a json-formatted string from the response."""
    with start_action(action_type='marshal', url=response.url):
        return MarshaledResponse(string=response.json(), from_url=response.url)


def store(marshaled_response: MarshaledResponse, db: TinyDB) -> None:
    """Persist the json-encoded string to the database."""
    with start_action(action_type='store'):
        db.insert(marshaled_response._asdict())
        

def main(*urls: T.List[str]) -> None:
    """Download, marshal, and store the content of each url."""
    
    db = TinyDB('db.json')
                
    with start_action(action_type='download_marshal_and_store', urls=urls):
        for url in urls:
            try:
                response = download(url)
                json_string = marshal(response)
                store(json_string, db)
            except:
                continue
    
```


```python
main(
    'https://httpbin.org/uuid',
    'http://google.com',
    'derp'
)
```

## Decorator example

Here we have an example of a decorator that could wrap any function and provide some logging functionality to it


```python
def logify(func):
    """Wrap a function in a eliot logger."""
    def wrapper(*args, **kwargs):
        with start_action(action_type=func.__name__, args=args, **kwargs):
            return func(*args, **kwargs)
    return wrapper


@logify
def reverse_string(string, **kwargs):
    return string[::-1]

reverse_string('hello', useless_kwarg="I'm useless")
reverse_string(-1)
```


    ---------------------------------------------------------------------------

    TypeError                                 Traceback (most recent call last)

    <ipython-input-4-2984a3506a49> in <module>()
         12 
         13 reverse_string('hello', useless_kwarg="I'm useless")
    ---> 14 reverse_string(-1)
    

    <ipython-input-4-2984a3506a49> in wrapper(*args, **kwargs)
          3     def wrapper(*args, **kwargs):
          4         with start_action(action_type=func.__name__, args=args, **kwargs):
    ----> 5             return func(*args, **kwargs)
          6     return wrapper
          7 


    <ipython-input-4-2984a3506a49> in reverse_string(string, **kwargs)
          9 @logify
         10 def reverse_string(string, **kwargs):
    ---> 11     return string[::-1]
         12 
         13 reverse_string('hello', useless_kwarg="I'm useless")


    TypeError: 'int' object is not subscriptable



```python
import time
import random

urls = [
    'https://httpbin.org/user-agent',
    'https://httpbin.org/ip',
    'https://httpbin.org/uuid',
    'https://httpbin.org/headers',
    'https://httpbin.org/get',
    'https://httpbin.org/robots.txt',
    'https://httpbin.org/xml',
    'http://google.com',
    'derp'
]

while True:
    time.sleep(3/random.randrange(1, 6))
    main(*random.choices(urls, k=random.randrange(1, 4)))
    
```
