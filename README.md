# OpenPAYGO - Python Lib

This repository contains the Python library for using implementing the different OpenPAYGOToken Suite technologies on your server (for generating tokens and decoding openpaygo metrics payloads) or device (for decoding tokens and making openpaygo metrics payloads).  

<p align="center">
  <img
    alt="Project Status"
    src="https://img.shields.io/badge/Project%20Status-beta-orange"
  >
  <a href="https://github.com/openpaygo/metrics/blob/main/LICENSE" target="_blank">
    <img
      alt="License"
      src="https://img.shields.io/github/license/openpaygo/openpaygo-python"
    >
  </a>
</p>

This open-source project was sponsored by: 
- Solaris Offgrid
- EnAccess

## Table of Contents

  - [Key Features](#key-features)
  - [Installing the library](#installing-the-library)
  - [Getting Started - OpenPAYGO Token](#getting-started---openpaygo-token)
    - [Generating Tokens](#generating-tokens)
    - [Decoding Tokens](#decoding-tokens)
  - [Getting Started - OpenPAYGO Metrics](#getting-started---openpaygo-metrics)
  - [Changelog](#changelog)
    - [2023-10-03 - v0.2.0](#2023-10-03---v020)

## Key Features
- Implements token generation and decoding with full support for the v2.3 of the [OpenPAYGO Token](https://github.com/EnAccess/OpenPAYGO-Token) specifications. 
- Implements payload authentication (verification + signing) and conversion from simple to condensed payload (and back) with full support of the v1.0-rc1 of the [OpenPAYGO Metrics](https://github.com/openpaygo/metrics) specifications. 

## Installing the library

You can install the library by running `pip install openpaygo` or adding `openpaygo` in your requirements.txt file and running `pip install -r requirements.txt`. 

## Getting Started - OpenPAYGO Token

### Generating Tokens

You can use the `generate_token()` function to generate an OpenPAYGOToken Token. The function takes the following parameters, and they should match the configuration in the hardware of the device: 

- `secret_key` (required): The secret key of the device. Must be passed as an hexadecimal string (with 32 characters). 
- `count` (required): The token count used to make the last token.
- `value` (optional): The value to be passed in the token (typically the number of days of activation). Optional if the `token_type` is Disable PAYG or Counter Sync. The value must be between 0 and 995. 
- `token_type` (optional): Used to set the type of token (default is Add Time). Token types can be found in the `TokenType` class: ADD_TIME, SET_TIME, DISABLE_PAYG, COUNTER_SYNC. 
- `starting_code` (optional): If not provided, it is generated according to the method defined in the standard (SipHash-2-4 of the key, transformed to digit by the same method as the token generation).
- `value_divider` (optional): The dividing factor used for the value. 
- `restricted_digit_set` (optional): If set to `true`, the the restricted digit set will be used (only digits from 1 to 4). 
- `extended_token` (optional): If set to `true` then a larger token will be generated, able to contain values up to 999999. This is for special use cases of each device, such as settings change, and is not set in the standard. 

The function returns the `updated_count` as a number as well as the `token` as a string, in that order. The function will raise a `ValueError` if the key is in the wrong format or the value invalid. 


**Example 1 - Add 7 days:**

```python
from openpaygo import generate_token
from myexampleproject import device_getter

# We get a device with the parameters we need from our database, this will be specific to your project
device = device_getter(serial=1234)

# We get the new token and update the count
device.count, new_token = generate_token(
  secret_key=device.secret_key,
  value=7,
  count=device.count
)

print('Token: '+new_token)
device.save() # We save the new count that we set for the device
```

**Example 2 - Disable PAYG (unlock forever):**

```python
from openpaygo import generate_token, TokenType

...

# We get the new token and update the count
device.count, new_token = generate_token(
  secret_key=device.secret_key,
  token_type=TokenType.DISABLE_PAYG,
  count=device.count
)

print('Token: '+new_token)
device.save() # We save the new count that we set for the device
```


### Decoding Tokens

You can use the `decode_token()` function to generate an OpenPAYGOToken Token. The function takes the following parameters, and they should match the configuration in the hardware of the device: 

- `token` (required): The token that was given by the user
- `secret_key` (required): The secret key of the device
- `count` (required): The token count of the last valid token. When a device is new, this is 1. 
- `used_counts` (optional): An array of recently used token counts, as returned by the function itself after the last valid token was decoded. This allows for handling unordered token entry. 
- `starting_code` (optional): If not provided, it is generated according to the method defined in the standard (SipHash-2-4 of the key, transformed to digit by the same method as the token generation).
- `value_divider` (optional): The dividing factor used for the value. 
- `restricted_digit_set` (optional): If set to `true`, the the restricted digit set will be used (only digits from 1 to 4). 


The function returns the following variable in this order: 
- `value`: The value associated with the token (if the token is ADD_TIME or SET_TIME). 
- `token_type`: The type of the token that was provided. Token types can be found in the `TokenType` class: ADD_TIME, SET_TIME, DISABLE_PAYG, COUNTER_SYNC or ALREADY_USED (if the token is valid but already used), INVALID (if the token was invalid). 
- `updated_count`: The token count of the token, if it was valid. 
- `updated_used_counts`: The updated array of recently used token, if the token was valid. 

The function will raise a `ValueError` if the key is in the wrong format, but will not raise an error if the token is invalid (as it is a common expected behaviour), to check the validity of the token you must check the return `token_type` and proceed accordingly depending on the type of token. 


**Example:**

```python
from openpaygo import decode_token

# We assume the users enters a token and that the device state is saved in my_device_state
...

# We decode the token
value, token_type, updated_count, updated_used_counts = decode_token(
  token=token_input,
  secret_key=my_device_state.secret_key,
  count=my_device_state.count,
  used_counts=my_device_state.used_counts
)

# If the token is valid, we update our count in the device state
if token_type not in [TokenType.ALREADY_USED, TokenType.INVALID]:
  my_device_state.count = updated_count
  my_device_state.used_counts = updated_used_counts

# We perform the appropriate behaviour based on the token data
if token_type == TokenType.ADD_TIME:
  my_device_state.days_remaining += value
  print(f'Added {value} days remaining')
elif token_type == TokenType.SET_TIME:
  my_device_state.days_remaining = value
  print(f'Set to {value} days remaining')
elif token_type == TokenType.DISABLE_PAYG:
  print('Unlocked Device')
  my_device_state.unlocked_forever = True
elif token_type == TokenType.COUNTER_SYNC:
  print('Counter Synced')
elif token_type == TokenType.ALREADY_USED:
  print('Token was already used')
elif token_type == TokenType.INVALID:
  print('Token is invalid')
```


## Getting Started - OpenPAYGO Metrics


You can use the `MetricsHandler` object to process your OpenPAYGO Metrics request from start to finish. It accepts the following initial inputs: 
- `metrics_payload` (required): The OpenPAYGO Metrics payload, in a JSON string format. 
- `secret_key` (optional): The secret key provided as a string containing 32 hexadecimal characters
- `data_format` (optional): The data format, provided as dictionnary matching the data format object specifications. 

It provides the following methods:
- `get_device_serial()`: Returns the serial number of the device as a string. 
- `set_device_parameters(secret_key, data_format)`: Used to set the device data required for proper processing of the request in the handler if it was not set initially, which is often the case as the serial number is usually required to fetch that data. It will return `ValueError` if either of the parameters is invalid. 
- `is_auth_valid()`: Returns `true` if the authentication provided is valid or `false` if not. 
- `get_simple_metrics()`: Returns the metrics provided in the simple expanded format. It will also convert relative timestamps into explicit timestamps for easier processing. 
- `expects_token_answer()`: Return `true` if the payload requested tokens in the answer. You can set the tokens to be returned by calling `add_tokens_to_answer(token_list)` with `token_list` being a list of token strings. 
- `expects_time_answer()`: Return `true` if the payload requested either relative time or absolute time in the answer. You can set the time to be returned by calling `add_time_to_answer(target_datetime)` with `target_datetime` being a datetime object. The function will automatically provide it in the correct format based on the request.  
- `add_settings_to_answer(settings_dict)`: Will add the provided settings dictionnary to the answer. 
- `add_extra_data_to_answer(extra_data_dict)`: Will add the provided extra data dictionnary to the answer. 
- `get_answer()`: Will return the answer as a string based on the request and the data added to answer, it will automatically handle the authentication and fomatting. 


**Example - Full Request flow:**

```python
from openpaygo import MetricsHandler
from my_db_service import get_device, store_metric


@app.route('/dd')
def device_data():
  # We load the metrics
  try:
    metrics = MetricsHandler(request.data)
  except ValueError as e:
    return {'error': 'Invalid data format'}, 400
  # We get the serial number and load the device data from our storage
  device = get_device(serial=metrics.get_device_serial())
  # We set the device parameters in the metrics handler
  metrics.set_device_parameters(
    secret_key=device.secret_key,
    data_format=device.data_format
  )
  # We check the authentication
  if not metrics.is_auth_valid():
    return {'error': 'Invalid authentication'}, 403
  # We transform the condensed data received from the device in simple data
  simple_data = metrics.get_simple_metrics()
  # We store the metrics in our database
  for metric_data in simple_data.get('data'):
    store_metric(name=metric_data['name'], value=metric_data['value'])
  for time_step in simple_data.get('historical_data'):
    for historical_metric_data in time_step:
      store_metric(name=metric_data['name'], value=metric_data['value'], time=time_step['timestamp'])
  # We prepare the answer
  if metrics.expects_token_answer():
    metrics.add_tokens_to_answer(device.pending_token_list)
  elif metrics.expects_time_answer():
    metrics.add_time_to_answer(device.expiration_datetime)
  # We can add extra data
  metrics.add_settings_to_answer({'language': 'fr-FR'})
  # The handler handles the signature, etc.
  return metrics.get_answer(), 200
```


## Changelog

### 2023-10-03 - v0.2.0
- First working version published on PyPI
- Has support for OpenPAYGO Token
- Has working CI for pushing to PyPI