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
| [401](https://brightdata.com/faqs/proxy-errors/error-401-how-to-avoid) | Unauthorized | Check your API key |
| [403](https://brightdata.com/faqs/proxy-errors/403-status-error-how-to-avoid) | Forbidden | You cannot access this data |
| 404 | Not Found | Site/Endpoint doesn’t exist |
| [408](https://brightdata.com/faqs/proxy-errors/error-408-how-to-avoid) | Request Timeout | Request timed out, try again |
| [429](https://brightdata.com/faqs/proxy-errors/429-error-how-to-avoid) | Too Many Requests | Slow down your requests |
| 500 | Internal Server Error | Generic server error, retry request |
| 501 | Not Implemented | Server doesn’t support this yet |
| [502](https://brightdata.com/faqs/proxy-errors/502-error-how-to-avoid) | Bad Gateway | Failed response from an upstream server |
| [503](https://brightdata.com/faqs/proxy-errors/503-error-how-to-avoid) | Service Unavailable | Server is temporarily down, retry later |
| [504](https://brightdata.com/faqs/proxy-errors/504-error-how-to-avoid) | Gateway Timeout | Timed out waiting for an upstream server |

## Retry Strategies

When implementing a retry mechanism in Python, you can leverage pre-built libraries like `HTTPAdapter` and `Tenacity`. Alternatively, you may choose to develop custom retry logic based on your specific needs.

A well-designed retry strategy should include both a retry limit and a backoff mechanism. The retry limit prevents infinite loops, ensuring that failed requests don’t continue indefinitely. A backoff strategy, which gradually increases the delay between retries, helps prevent excessive requests that could lead to being blocked or overloading the server.

- **Retry Limits**: It’s essential to define a retry limit. After a specified number of attempts (X), the scraper should stop retrying to avoid infinite loops.  
- **Backoff Algorithm**: A gradual increase in wait time between retries helps prevent overwhelming the server. Start with a small delay, such as 0.3 seconds, then incrementally increase it to 0.6 seconds, 1.2 seconds, and so forth.

## HTTPAdapter

With `HTTPAdapter`, we need to configure three things: `total`, `backoff_factor`, and `status_forcelist`. `allowed_methods` isn’t a requirement per se, but it helps define our retry conditions and thus makes our code safer. In the code below, we use [httpbin](https://httpbin.org/) to automatically force an error and trigger the retry logic.

```python
import logging
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

# Configure logging
logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")
logger = logging.getLogger(__name__)

# Create a session
session = requests.Session()

# Configure retry settings
retry = Retry(
    total=3,  # Maximum retries
    backoff_factor=0.3,  # Time between retries (exponential backoff)
    status_forcelist=(429, 500, 502, 503, 504),  # Status codes to trigger a retry
    allowed_methods={"GET", "POST"}  # Allow retries for GET and POST
)

# Mount the adapter with our custom settings
adapter = HTTPAdapter(max_retries=retry)
session.mount("http://", adapter)
session.mount("https://", adapter)

# Function to make a request and test retry logic
def make_request(url, method="GET"):
    try:
        logger.info(f"Making a {method} request to {url} with retry logic...")
        
        if method == "GET":
            response = session.get(url)
        elif method == "POST":
            response = session.post(url)
        else:
            logger.error("Unsupported HTTP method: %s", method)
            return
        
        response.raise_for_status()
        logger.info("✅ Request successful: %s", response.status_code)
    
    except requests.exceptions.RequestException as e:
        logger.error("❌ Request failed after retries: %s", e)
        logger.info("Retries attempted: %d", len(response.history) if response else 0)

# Test Cases
make_request("https://httpbin.org/status/200")  # ✅ Should succeed without retries
make_request("https://httpbin.org/status/500")  # ❌ Should retry 3 times and fail
make_request("https://httpbin.org/status/404")  # ❌ Should fail immediately (no retries)
make_request("https://httpbin.org/status/500", method="POST")  # ❌ Should retry 3 times and fail
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
2024-06-10 12:00:00 - INFO - Making a GET request to https://httpbin.org/status/200 with retry logic...
2024-06-10 12:00:00 - INFO - ✅ Request successful: 200

2024-06-10 12:00:01 - INFO - Making a GET request to https://httpbin.org/status/500 with retry logic...
2024-06-10 12:00:02 - ERROR - ❌ Request failed after retries: 500 Server Error: INTERNAL SERVER ERROR for url: ...
2024-06-10 12:00:02 - INFO - Retries attempted: 3

2024-06-10 12:00:03 - INFO - Making a GET request to https://httpbin.org/status/404 with retry logic...
2024-06-10 12:00:03 - ERROR - ❌ Request failed after retries: 404 Client Error: NOT FOUND for url: ...
2024-06-10 12:00:03 - INFO - Retries attempted: 0

2024-06-10 12:00:04 - INFO - Making a POST request to https://httpbin.org/status/500 with retry logic...
2024-06-10 12:00:05 - ERROR - ❌ Request failed after retries: 500 Server Error: INTERNAL SERVER ERROR for url: ...
2024-06-10 12:00:05 - INFO - Retries attempted: 3
```

## Tenacity

You can also use [`Tenacity`](https://tenacity.readthedocs.io/en/latest/), a popular open source retry library for Python. It’s not limited to HTTP, but it gives you an expressive way to implement retries.

Start with installing `Tenacity`:

```bash
pip install tenacity
```

Once installed, create a _decorator_ and use it to wrap a requests function. With the `@retry` decorator, add the `stop`, `wait`, and `retry` arguments.

```python
import logging
import requests
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type, retry_if_result, RetryError

# Configure logging
logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")
logger = logging.getLogger(__name__)

# Define a retry strategy
@retry(
    stop=stop_after_attempt(3),  # Retry up to 3 times
    wait=wait_exponential(multiplier=0.3),  # Exponential backoff
    retry=(
        retry_if_exception_type(requests.exceptions.RequestException) |  # Retry on request failures
        retry_if_result(lambda r: r.status_code in {500, 502, 503, 504})  # Retry on specific HTTP status codes
    ),
)
def make_request(url):
    logger.info("Making a request with retry logic to %s...", url)
    response = requests.get(url)
    response.raise_for_status()
    logger.info("✅ Request successful: %s", response.status_code)
    return response

# Attempt to make the request
try:
    make_request("https://httpbin.org/status/500")  # Test with a failing status code
except RetryError as e:
    logger.error("❌ Request failed after all retries: %s", e)    
```

The logic and settings here are very similar to the first example with `HTTPAdapter`:

- `stop=stop_after_attempt(3)`: This tells `tenacity` to give up after 3 failed retries.
- `wait=wait_exponential(multiplier=0.3)` uses the same wait that we used before. It also backs off exponentially, just like before.
- `retry=retry_if_exception_type(requests.exceptions.RequestException)` tells `tenacity` to use this logic every time a `RequestException` occurs.
- `make_request()` makes a request to our error endpoint. It receives all of the traits from the decorator you created above it.

When you run this code, you get a similar output:

```
2024-06-10 12:00:00 - INFO - Making a request with retry logic to https://httpbin.org/status/500...
2024-06-10 12:00:01 - WARNING - Retrying after 0.3 seconds...
2024-06-10 12:00:01 - INFO - Making a request with retry logic to https://httpbin.org/status/500...
2024-06-10 12:00:02 - WARNING - Retrying after 0.6 seconds...
2024-06-10 12:00:02 - INFO - Making a request with retry logic to https://httpbin.org/status/500...
2024-06-10 12:00:03 - ERROR - ❌ Request failed after all retries: RetryError[...]
```

## Building a Custom Retry Mechanism

You can also create a custom retry mechanism, which is often the best approach when working with specialized code. With a relatively small amount of code, you can achieve the same functionality provided by existing libraries while tailoring it to your specific needs.

The code below demonstrates how to import `sleep` for the exponential backoff, set the configuration (`total`, `backoff_factor` and `bad_codes`), and use a `while` loop to hold the retry logic. `while`you still have tries and you haven’t succeeded, attempt the request.

```python
import logging
import requests
from time import sleep

# Configure logging
logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")
logger = logging.getLogger(__name__)

# Create a session
session = requests.Session()

# Define retry settings
TOTAL_RETRIES = 3
INITIAL_BACKOFF = 0.3
BAD_CODES = {429, 500, 502, 503, 504}

def make_request(url):
    current_tries = 0
    backoff = INITIAL_BACKOFF
    success = False

    while current_tries < TOTAL_RETRIES and not success:
        try:
            logger.info("Making a request with retry logic to %s...", url)
            response = session.get(url)
            
            if response.status_code in BAD_CODES:
                raise requests.exceptions.HTTPError(f"Received {response.status_code}, triggering retry")
            
            response.raise_for_status()
            logger.info("✅ Request successful: %s", response.status_code)
            success = True
            return response

        except requests.exceptions.RequestException as e:
            logger.error("❌ Request failed: %s, retries left: %d", e, TOTAL_RETRIES - current_tries - 1)
            if current_tries < TOTAL_RETRIES - 1:
                logger.info("⏳ Retrying in %.1f seconds...", backoff)
                sleep(backoff)
                backoff *= 2  # Exponential backoff
            current_tries += 1

    logger.error("🚨 Request failed after all retries.")
    return None

# Test Cases
make_request("https://httpbin.org/status/500")  # ❌ Should retry 3 times and fail
make_request("https://httpbin.org/status/200")  # ✅ Should succeed without retries
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
2024-06-10 12:00:00 - INFO - Making a request with retry logic to https://httpbin.org/status/500...
2024-06-10 12:00:01 - ERROR - ❌ Request failed: Received 500, triggering retry, retries left: 2
2024-06-10 12:00:01 - INFO - ⏳ Retrying in 0.3 seconds...
2024-06-10 12:00:02 - INFO - Making a request with retry logic to https://httpbin.org/status/500...
2024-06-10 12:00:03 - ERROR - ❌ Request failed: Received 500, triggering retry, retries left: 1
2024-06-10 12:00:03 - INFO - ⏳ Retrying in 0.6 seconds...
2024-06-10 12:00:04 - INFO - Making a request with retry logic to https://httpbin.org/status/500...
2024-06-10 12:00:05 - ERROR - ❌ Request failed: Received 500, triggering retry, retries left: 0
2024-06-10 12:00:05 - ERROR - 🚨 Request failed after all retries.
```

## Conclusion

To avoid all kinds of failed requests, we’ve developed products like the [Web Unlocker API](https://brightdata.com/products/web-unlocker) and [Scraping Browser](https://brightdata.com/products/scraping-browser). These tools automatically handle anti-bot measures, CAPTCHA challenges, and IP blocks, ensuring seamless and efficient web scraping for even the most challenging websites.

Sign up now and start your free trial today.
