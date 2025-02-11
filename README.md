# Managing Failed Requests in Python

[![Promo](https://github.com/luminati-io/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.com/) 

This guide explains how to handle failed HTTP requests in Python with retry strategies and custom logic.

- [What Are Status Codes?](#what-are-status-codes)
- [Retry Strategies](#retry-strategies)
- [HTTPAdapter](#httpadapter)
- [Tenacity](#tenacity)
- [Building a Custom Retry Mechanism](#building-a-custom-retry-mechanism)
- [Conclusion](#conclusion)

## What Are Status Codes?

Status codes are standardized three-digit numbers used in various protocols to indicate the result of a request. According to [Mozilla](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status), HTTP status codes can be broken down into the following categories:

- **100-199**: Informational responses
- **200-299**: Successful responses
- **300-399**: Redirection messages
- **400-499**: Client error messages
- **500-599**: Server error messages

When developing client-side applications like web scrapers, it's crucial to pay attention to status codes in the 400 and 500 ranges. Codes in the 400s typically indicate client-side errors, such as authentication failures, rate limiting, timeouts, or the well-known _404: Not Found error_. Meanwhile, status codes in the 500s signal server-side issues that may require retries or alternative handling strategies.

Here is a list of common error codes (taken from Mozilla’s [official documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status#client_error_responses)) you will encounter when performing web scraping:

| **Status Code** | **Meaning** | **Description** |
| --- | --- | --- |
| 400 | Bad Request | Check your request format |
| 401 | Unauthorized | Check your API key |
| 403 | Forbidden | You cannot access this data |
| 404 | Not Found | Site/Endpoint doesn’t exist |
| 408 | Request Timeout | Request timed out, try again |
| 429 | Too Many Requests | Slow down your requests |
| 500 | Internal Server Error | Generic server error, retry request |
| 501 | Not Implemented | Server doesn’t support this yet |
| 502 | Bad Gateway | Failed response from an upstream server |
| 503 | Service Unavailable | Server is temporarily down, retry later |
| 504 | Gateway Timeout | Timed out waiting for an upstream server |

## Retry Strategies

When implementing a retry mechanism in Python, you can leverage pre-built libraries like `HTTPAdapter` and `Tenacity`. Alternatively, you may choose to develop custom retry logic based on your specific needs.

A well-designed retry strategy should include both a retry limit and a backoff mechanism. The retry limit prevents infinite loops, ensuring that failed requests don’t continue indefinitely. A backoff strategy, which gradually increases the delay between retries, helps prevent excessive requests that could lead to being blocked or overloading the server.

- **Retry Limits**: It’s essential to define a retry limit. After a specified number of attempts (X), the scraper should stop retrying to avoid infinite loops.  
- **Backoff Algorithm**: A gradual increase in wait time between retries helps prevent overwhelming the server. Start with a small delay, such as 0.3 seconds, then incrementally increase it to 0.6 seconds, 1.2 seconds, and so forth.

## HTTPAdapter

With `HTTPAdapter`, we need to configure three things: `total`, `backoff_factor`, and `status_forcelist`. `allowed_methods` isn’t a requirement per se, but it helps define our retry conditions and thus makes our code safer. In the code below, we use [httpbin](https://httpbin.org/) to automatically force an error and trigger the retry logic.

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

#create a session
session = requests.Session()

#configure retry settings
retry = Retry(
    total=3,                  #maximum retries
    backoff_factor=0.3,       #time between retries (exponential backoff)
    status_forcelist=(429, 500, 502, 503, 504), #status codes to trigger a retry
    allowed_methods={"GET", "POST"}
)

#mount the adapter with our custom settings
adapter = HTTPAdapter(max_retries=retry)
session.mount("http://", adapter)
session.mount("https://", adapter)

#actually make a request with our retry logic
try:
    print("Making a request with retry logic...")
    response = session.get("https://httpbin.org/status/500")
    response.raise_for_status()
    print("✅ Request successful:", response.status_code)
except requests.exceptions.RequestException as e:
    print("❌ Request failed after retries:", e)
```

Once you created a `Session` object, do this:

- Create a `Retry` object and define:
    - `total`: The maximum limit for retrying a request.
    - `backoff_factor`: Time to wait between retries. This adjusts exponentially as our retries increase.
    - `status_forcelist`: A list of bad status codes. Any codes in this list will automatically trigger a retry.
- Create an `HTTPAdapter` object with our `retry` variable: `adapter = HTTPAdapter(max_retries=retry)`.
- Once you’ve created the `adapter`, mount it to the HTTP and HTTPS methods using `session.mount()`.

When you run this code, the three retries (`total=3`) will run, and then you’ll get the following output.

```
Making a request with retry logic...
❌ Request failed after retries: HTTPSConnectionPool(host='httpbin.org', port=443): Max retries exceeded with url: /status/500 (Caused by ResponseError('too many 500 error responses'))
```

## Tenacity

You can also use [`Tenacity`](https://tenacity.readthedocs.io/en/latest/), a popular open source retry library for Python. It’s not limited to HTTP, but it gives you an expressive way to implement retries.

Start with installing `Tenacity`:

```bash
pip install tenacity
```

Once installed, create a _decorator_ and use it to wrap a requests function. With the `@retry` decorator, add the `stop`, `wait`, and `retry` arguments.

```python
import requests
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type, RetryError

#define a retry strategy
@retry(
    stop=stop_after_attempt(3),  #retry up to 3 times
    wait=wait_exponential(multiplier=0.3),  #exponential backoff
    retry=retry_if_exception_type(requests.exceptions.RequestException),  #retry on request failures
)

def make_request():
    print("Making a request with retry logic...")
    response = requests.get("https://httpbin.org/status/500")
    response.raise_for_status()
    print("✅ Request successful:", response.status_code)
    return response

# Attempt to make the request
try:
    make_request()
except RetryError as e:
    print("❌ Request failed after all retries:", e)

```

The logic and settings here are very similar to the first example with `HTTPAdapter`:

- `stop=stop_after_attempt(3)`: This tells `tenacity` to give up after 3 failed retries.
- `wait=wait_exponential(multiplier=0.3)` uses the same wait that we used before. It also backs off exponentially, just like before.
- `retry=retry_if_exception_type(requests.exceptions.RequestException)` tells `tenacity` to use this logic every time a `RequestException` occurs.
- `make_request()` makes a request to our error endpoint. It receives all of the traits from the decorator you created above it.

When you run this code, you get a similar output:

```
Making a request with retry logic...
Making a request with retry logic...
Making a request with retry logic...
❌ Request failed after all retries: RetryError[<Future at 0x75e762970760 state=finished raised HTTPError>]
```

## Building a Custom Retry Mechanism

You can also create a custom retry mechanism, which is often the best approach when working with specialized code. With a relatively small amount of code, you can achieve the same functionality provided by existing libraries while tailoring it to your specific needs.

The code below demonstrates how to import `sleep` for the exponential backoff, set the configuration (`total`, `backoff_factor` and `bad_codes`), and use a `while` loop to hold the retry logic. `while`you still have tries and you haven’t succeeded, attempt the request.

```python
import requests
from time import sleep

#create a session
session = requests.Session()

#define our retry settings
total = 3
backoff_factor = 0.3
bad_codes = [429, 500, 502, 503, 504]

#try counter and success boolean
current_tries = 0
success = False

#attempt until we succeed or run out of tries
while current_tries < total and not success:
    try:
        print("Making a request with retry logic...")
        response = session.get("https://httpbin.org/status/500")
        if response.status_code in bad_codes:
            raise requests.exceptions.HTTPError(f"Received {response.status_code}, triggering retry")
        print("✅ Request successful:", response.status_code)
        success = True
    except requests.exceptions.RequestException as e:
        print(f"❌ Request failed: {e}, retries left: {total-current_tries}")
        sleep(backoff_factor)
        backoff_factor = backoff_factor * 2
        current_tries+=1
```

The actual logic here is handled by a simple `while` loop.

- If `response.status_code` is in the list of `bad_codes`, the script throws an exception.
- If a request fails, the script:
    - Prints an error message to the console.
    - `sleep(backoff_factor)` waits before sending the next request.
    - `backoff_factor = backoff_factor * 2` doubles our `backoff_factor` for the next try.
    - Increments `current_tries` so it doesn’t stay in the loop indefinitely.

Here’s the output from the custom retry code.

```
Making a request with retry logic...
❌ Request failed: Received 500, triggering retry, retries left: 3
Making a request with retry logic...
❌ Request failed: Received 500, triggering retry, retries left: 2
Making a request with retry logic...
❌ Request failed: Received 500, triggering retry, retries left: 1
```

## Conclusion

To avoid all kinds of failed requests, we’ve developed products like the [Web Unlocker API](https://brightdata.com/products/web-unlocker) and [Scraping Browser](https://brightdata.com/products/scraping-browser). These tools automatically handle anti-bot measures, CAPTCHA challenges, and IP blocks, ensuring seamless and efficient web scraping for even the most challenging websites.

Sign up now and start your free trial today.