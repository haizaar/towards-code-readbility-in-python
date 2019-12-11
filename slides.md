---
title: Towards Code Readability in Python
author: Zaar Hai
...

# Let's begin!

## Talk summary

A collection of semi-related topics about good-looking code.
Highly opinionated.

Principles:

* You should understand a code in a file without reading the rest of the project
* Don't write comments for obvious things
* In fact, don't write comments at all unless READY need to
* Use language tools themselves to improve clarity

# Oh, those constants

## We've all seen this

```python
if status == "ready":
	...

```

##  Then we've tried to fix it

```python
class StatusType:
	ready = "ready"
	in_prog = "in_progress"

if status == StatusType.ready:
	...

```

. . .


Issues:

* No membership test
* Can't see all of the statuses

. . .

Hard to write code like:

```python
status = open(...).read()
if status not in StatusType:
	raise ValueError(f"Invalid status {status}. Expected {StatusType.all()}")

```

## Dotted dict gets you half-way:

```python
from dotted_dict import DottedDict  # pip install dotted-dict

StatusType = DottedDict({
	"ready": "ready",
	"in_prog": "in_pogress",
})

status = open(...).read()  # in_progress
if status not in StatusType:  # <-- Fail!
	raise ValueError(f"Invalid status {status}. Expected {StatusType.all()}")
```

. . .

Correction:

```python
status = open(...).read()  # in_progress
if status not in StatusType.values():
	raise ValueError(f"Invalid status {status}. Expected {StatusType.all()}")
```

## Half way indeed

Looks quite unclean and fragile. We also loose strong types. E.g.

```python
StatusType = DottedDict({
	"ready": "ready",
	"in_prog": "in_pogress",
})

AnotherStatusType = DottedDict({
	"ready": "ready",
	"in_prog": "in_pogress",
})

StatusType.ready == AnotherStatusType.ready  # True
```

# Enums to the rescue

## Enums will do you good

```python
from enum import Enum  # stdlib

class StatusType(Enum):
	ready = "ready"
	in_prog = "in_progress"

repr(StatusType.in_prog)  # <StatusType.in_prog: 'in_progress'>
StatusType.in_prog.value  # in_progress
StatusType.in_prog.name   # in_prog

logger.debug("All types are", type=[e.value for e in StatusType])
```
. . .

```python
status: StatusType(open(...).read())  # in_progress

if status is StatusType.ready:
	...
elif status is StatusType.in_prog:
	...
```

# Caught the strings - let's catch some dicts

## Raw dicts are brain buggers

```python
def status_progress() {
	"""Check status file and report total progress between 0 to 100"""
	sdata = json.load(open(...))

	if sdata["current_stage"]:
		return 100

	progress = 0
	for step in sdata["data"]:
		progress += step[1] / len(sdata["data"])

	return progress
```

**No way** you can really understand the code without *guessing* the input format.

## Comments do not help

```python
def status_progress():
	"""Check status file and report total progress between 0 to 100"""

	# Expecting the following JSON input
	#  {
	#    "current_stage": "installing",
	#    "data": [
	#        ["unpacking", 100],
	#        ["installing", 37]
	#    ]
	#  }
	#
	sdata = json.load(open(...))

	if sdata["current_stage"]:
		return 100

	progress = 0
	for step in sdata["data"]:
		progress += step[1] / len(sdata["data"])

	return progress
```

## Modelling to the rescue

```python

from enum import Enum

class StageType(Enum):
	unpacking = "unpacking"
	installing = "installing"
```
. . .
```python
from pydantic import BaseModel, ValidationError
from typing import Optional, Tuple

class Status(BaseModel):
	current_stage: Optional[StageType]
	data: Tuple[Tuple[StageName, int]]
```

## Modelling to the rescue

```python
class Status(BaseModel):
	current_stage: Optional[StageType]
	data: Tuple[Tuple[StageName, int]]


def status_progress() -> int:
	status_info = json.load(open(...))
	try:
		status = Status(**status_info)
	except ValidationError:
		logger.exception("Failed to parse data", data=status_info)
		raise

	if status.current_stage:
		return 100

	progress = 0
	for step in status.data:
		progress += step[1] / len(sdata["data"])

	return progress
```

## Modelling improved

```python
from pydantic import BaseModel, ValidationError
from typing import NewType, Optional, Tuple

ProgressPct = NewType("ProgressPct", int)

class Status(BaseModel):
	current_stage: Optional[StageType]
	data: Tuple[Tuple[StageName, ProgressPct]]
```
. . .
```python
	def total_progress(self) -> ProgressPct:
		done_pct = sum(s[1] for s in self.data)
        return done_pct / len(self.data)
```

## No need for progress reporting function anymore!

```python
def get_status() -> Status:
	status_info = json.load(open(...))
	try:
		return Status(**status_info)
	except ValidationError:
		logger.exception("Failed to parse data", data=status_info)
		raise
```

## Icing on the cake

```python
from pydantic import BaseModel, Field

class Status(BaseModel):
	current_stage: Optional[StageType]
	stages: Tuple[Tuple[StageName, ProgressPct]] = Field(..., alias="data")

	...
```

# Composition

## Let's write rest API client

Let's say we have API that manages the following objects:

* Users: list, add, delete, suspend
* Servers: list, install, stop, start

How it's best to build Python *client* for that?

## Brute force approach

```python
import requests

def Client:
	def __init__(self, host, port=80):
		self.base_url = f"http://{host}:{port}/api/v1"
		self.session = requests.Session()
```
. . .
```python
	def list_servers(self):
		res = self.session.get(self.base_url + "/servers")
		res.raise_for_status()
		return res.json()  # <-- Dictionaries again :(
```
. . .
```python
	def install_server(self):
		...
	def stop_server(self):
		...
	def start_server(self):
		...

	def list_users(self):
		...
	def add_users(self):
		...
	def delete_user(self):
		...
```
Eventually will end up with huge class that is bloated with potentially hundreds of methods.

## Factor out HTTP client

```python
from dataclasses import dataclass, ClassVar
import requests
from yarl import URL

@dataclass
class HTTPClient:
	api_prefix: ClassVar[str] = "api/v1"

	base_url: URL
	session: requests.Session

	@classmethod
	def from_host_port(cls, host: str, port: int) -> HTTPClient:
		base_url = URL(f"http://{host}:{port}") / cls.api_prefix
		session = requests.Session()
		return cls(base_url=base_url, session=session)
```
. . .
```python
	def close(self) -> None:
		self.session.close()

	def __enter__(self) -> HTTPClient:
		return self

	def __exit__(self, *args, **kwargs) -> None:
		self.close()
```

## Factor out HTTP client

Let's add some useful functionality, should we?
```python
from typing import Any, Dict

@dataclass
class HTTPClient:
	api_prefix: ClassVar[str] = "api/v1"

	base_url: URL
	session: requests.Session

	@classmethod
	def from_host_port(cls, host: str, port: int) -> HTTPClient:
		base_url = URL(f"http://{host}:{port}") / cls.api_prefix
		session = requests.Session()
		return cls(base_url=base_url, session=session)

	...

	def request(self, method: Callable[..., requests.Response], subpath: str,
				query: Optional[dict] = None, data: Optional[dict] = None) -> Union[dict, list]:
		kwargs: Dict[str, Any] = {}
		if data:
			kwargs["json"] = data
		if query:
			kwargs["params"] = query

		url = self.base_url / subpath.lstrip("/")
		res = method(url, **kwargs)
		res.raise_for_status()
		return res.json()  # <-- Dictionaries still :(

	def get(self, *args: List[Any], **kwargs: Dict[str, Any]) -> Union[dict, list]:
		return self.request(self.session.get, *args, **kwargs)
```

## Back to API

Now we can compose code for the API client itself

```python
@dataclass
class APIClient:
	http: HTTPClient
```
. . .
```python
	@classmethod
	def from_host_port(cls, host: str, port: int) -> APIClient:
		http = HTTPClient.from_host_post(host, port)
		return cls(http=http)
```

## Lets add some meat

```python
@dataclass
class Servers:
	http: HTTPClient

	def list(self): pass

@dataclass
class Users:
	http: HTTPClient

	def list(self): pass
```
. . .
```python
@dataclass
class APIClient:
	http: HTTPClient
	servers: Servers
	users: Users

	@classmethod
	def from_host_port(cls, host: str, port: int) -> APIClient:
		http = HTTPClient.from_host_post(host, port)
		servers = Servers(http)
		users = Users(http)
		return cls(http=http, servers=servers, users=users)
```
. . .

So now we can just
```python
api = APIClieint.from_host_port(...)
api.servers.list()
api.users.list()
```
It's pluggable!

## Let's get rid of dicts, again

```python
from pydantic import BaseModel

class Server(BaseModel):
	...

@dataclass
class Server:
	http: HTTPClient

	def list(self) -> List[Server]:
		servers_info = self.http.get(...)
		return [Server(**si) for si in servers_info]
```

If you server uses Python/pydantic you can even share the models!

## And makes paths beautiful

```python
@dataclass
class Server:
	http: HTTPClient

	class paths:
		servers: str = "servers"
		stop: str = servers + "/{id}/stop"

	def list(self) -> List[Server]:
		servers_info = self.http.get(self.paths.servers)
		return [Server(**si) for si in servers_info]

	def stop(self, server: Server) -> Server:
		server_info = self.http.post(self.paths.stop.format(id=server.id))
server_info)
```

# Questions
