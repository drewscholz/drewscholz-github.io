# FoundationDB Lambda Client
---
*Python container lambda running FoundationDB client to set and get data from FoundationDB servers*
## 1. Dockerfile for Lambda Container
### Notes:

* Copies fdb.cluster file into the image
* Installs foundationdb and gevent
* gevent is used to workaround the python lambda parallel processing constraint
### Dockerfile:
```
# https://docs.aws.amazon.com/lambda/latest/dg/images-create.html#images-create-from-base
FROM ubuntu:20.04

# Install dependencies for Lambda and python3.9
RUN apt-get update && apt-get install -y wget g++ make cmake unzip libcurl4-openssl-dev
RUN apt-get install -y software-properties-common gcc && add-apt-repository -y ppa:deadsnakes/ppa
RUN apt-get update && apt-get install -y python3.9 python3-distutils python3-pip python3-apt

# Install FoundationDB Client
RUN wget -c https://github.com/apple/foundationdb/releases/download/7.2.5/foundationdb-clients_7.2.5-1_amd64.deb
RUN dpkg -i foundationdb-clients_7.2.5-1_amd64.deb

# Setup function directory
ARG FUNCTION_DIR="/function"
RUN mkdir -p ${FUNCTION_DIR}

# Install python packages
RUN python3.9 -m pip install --target ${FUNCTION_DIR} awslambdaric
RUN python3.9 -m pip install foundationdb==7.2.5
RUN python3.9 -m pip install gevent==22.10.2

# Set working directory to function root directory
WORKDIR ${FUNCTION_DIR}

ENV FDB_CLUSTER_FILE=/var/fdb/fdb.cluster
ENV FDB_NETWORKING_MODE=container

# Copy cluster file from FoundationDB server
COPY /path/to/fdb.cluster /var/fdb/fdb.cluster

COPY app/* ${FUNCTION_DIR}

ENTRYPOINT [ "python3.9", "-m", "awslambdaric" ]

CMD [ "handler.lambda_handler" ]
```

## 2. Lambda Handler
### Notes:
* the default event_mode for fdb.open() does not work in AWS Lambda, see **Lambda Contraints with Parallel Processing** below

### Python example:
```
import fdb

fdb.api_version(720)
db = fdb.open(event_model="gevent")

foobar_dir = fdb.directory.open(db, ('foo', 'bar'))
subspace_data = foobar_dir['data']


def lambda_handler(event, context):
    print("setting value")
    my_key = ('asdf', '1234')
    set_value(db, my_key, 'my_value')
    print("set value, getting value")
    print('value: ', get_value(db, my_key))


@fdb.transactional
def set_value(tr, key, value):
    tr.set(subspace_data.pack(key,), value.encode('utf-8'))


@fdb.transactional
def get_value(tr, key):
    value = tr.get(subspace_data.pack(key,))
    return value.decode('utf-8')


```

---

## **Lambda Contraints with Parallel Processing**
### Related documentation:
* https://aws.amazon.com/blogs/compute/parallel-processing-in-python-with-aws-lambda/
* https://stackoverflow.com/questions/34005930/multiprocessing-semlock-is-not-implemented-when-running-on-aws-lambda

### Python AWS Lambda Runtime Error:
```
[ERROR] OSError: [Errno 38] Function not implemented
Traceback (most recent call last):
  File "/function/handler.py", line 14, in lambda_handler
    datapoints_dir = fdb.directory.open(db, ('my_dir', 'my_sub_dir'))
  File "/usr/local/lib/python3.9/dist-packages/fdb/impl.py", line 282, in wrapper
    ret = func(*largs, **kwargs)
  File "/usr/local/lib/python3.9/dist-packages/fdb/directory_impl.py", line 305, in open
    return self._create_or_open_internal(tr, path, layer, allow_create=False)
  File "/usr/local/lib/python3.9/dist-packages/fdb/directory_impl.py", line 233, in _create_or_open_internal
    self._check_version(tr, write_access=False)
  File "/usr/local/lib/python3.9/dist-packages/fdb/directory_impl.py", line 452, in _check_version
    if not version.present():
  File "/usr/local/lib/python3.9/dist-packages/fdb/impl.py", line 922, in present
    return self.value is not None
  File "/usr/local/lib/python3.9/dist-packages/fdb/impl.py", line 785, in __get__
    return self.method(obj)
  File "/usr/local/lib/python3.9/dist-packages/fdb/impl.py", line 804, in value
    self.block_until_ready()
  File "/usr/local/lib/python3.9/dist-packages/fdb/impl.py", line 654, in block_until_ready
    semaphore = multiprocessing.Semaphore(0)
  File "/usr/lib/python3.9/multiprocessing/context.py", line 83, in Semaphore
    return Semaphore(value, ctx=self.get_context())
  File "/usr/lib/python3.9/multiprocessing/synchronize.py", line 126, in __init__
    SemLock.__init__(self, SEMAPHORE, value, SEM_VALUE_MAX, ctx=ctx)
  File "/usr/lib/python3.9/multiprocessing/synchronize.py", line 57, in __init__
    sl = self._semlock = _multiprocessing.SemLock(
```
---
## **Additional Notes**
* Configure lambda in the same VPC as the FoundationDB EC2 servers: https://docs.aws.amazon.com/lambda/latest/dg/configuration-vpc.html
* Configure EC2 servers security group inbound rules to allow from VPC
---
Thanks to https://dillinger.io/ for the styled html export
