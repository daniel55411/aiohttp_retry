# Simple aiohttp retry client

This package is similar to [Tornado-retry-client](https://github.com/wpjunior/tornado-retry-client). \
Python 3.6 or higher.

**Install**: `pip install aiohttp-retry`.

### Warning
This current version is 2.0. It hasn't backward compatibility for previous versions. \
You still can use [v1.2](https://github.com/inyutin/aiohttp_retry/tree/v1.2) (pip install aiohttp-retry==1.2), but it is unsupported.


### Examples of usage:
```python
from aiohttp_retry import RetryClient, RetryOptions

async def main():
    retry_options = RetryOptions(attempts=1)
    retry_client = RetryClient(raise_for_status=False, retry_options=retry_options)
    async with retry_client.get('https://ya.ru') as response:
        print(response.status)
        
    await retry_client.close()
```
```python
from aiohttp_retry import RetryClient, RetryOptions

async def main():
    retry_options = RetryOptions(attempts=1)
    retry_client = RetryClient(raise_for_status=False, retry_options=retry_options)

    response = await retry_client.get('/ping')
    print(response.status)
        
    await retry_client.close()
```

```python
from aiohttp_retry import RetryClient

async def main():
    async with RetryClient() as client:
        async with client.get('https://ya.ru') as response:
            print(response.status)
```

```python
import logging
import sys
from types import SimpleNamespace

from aiohttp import ClientSession, TraceConfig, TraceRequestStartParams

from aiohttp_retry import RetryClient, RetryOptions


handler = logging.StreamHandler(sys.stdout)
logging.basicConfig(handlers=[handler])
logger = logging.getLogger(__name__)
retry_options = RetryOptions(attempts=2)


async def on_request_start(
    session: ClientSession,
    trace_config_ctx: SimpleNamespace,
    params: TraceRequestStartParams,
) -> None:
    current_attempt = trace_config_ctx.trace_request_ctx['current_attempt']
    if retry_options.attempts <= current_attempt:
        logger.warning('Wow! We are in last attempt')


async def main():
    trace_config = TraceConfig()
    trace_config.on_request_start.append(on_request_start)
    retry_client = RetryClient(retry_options=retry_options, trace_configs=[trace_config])

    response = await retry_client.get('https://httpstat.us/503', ssl=False)
    print(response.status)

    await retry_client.close()
```
Look tests for more examples. \
Be aware: last request returns as it is.

### Documentation
`RetryClient` takes the same arguments as ClientSession[[docs](https://docs.aiohttp.org/en/stable/client_reference.html)] \
`RetryClient` has methods:
- get
- options
- head
- post
- put
- patch
- put
- delete

They are same as for `ClientSession`, but take one possible additional argument: 
```python
from typing import Optional, Set, Type

class RetryOptions:
    def __init__(
        self,
        attempts: int = 3,  # How many times we should retry
        start_timeout: float = 0.1,  # Base timeout time, then it exponentially grow
        max_timeout: float = 30.0,  # Max possible timeout between tries
        factor: float = 2.0,  # How much we increase timeout each time
        statuses: Optional[Set[int]] = None,  # On which statuses we should retry
        exceptions: Optional[Set[Type[Exception]]] = None,  # On which exceptions we should retry
    )
    ...
```
You can specify `RetryOptions` both for `RetryClient` and it's methods. 
`RetryOptions` in methods override `RetryOptions` defined in `RetryClient` constructor.

#### Request Trace Context
`RetryClient` add *current attempt number* to `request_trace_ctx` (see examples, 
for more info see [aiohttp doc](https://docs.aiohttp.org/en/stable/client_advanced.html#aiohttp-client-tracing)) 
