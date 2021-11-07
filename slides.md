### Monitor your Python tests to optimise your code!

Jean-Sébastien Dieu, Architect @ [CFM](https://www.cfm.fr)

jean-sebastien.dieu@cfm.fr

--- ---

##### Context

- You are a Python developer.
- Expectations: 
   - Implement a resource critical function (primality check)
   - Minimum memory consumption
   - A call does not last more than a few milliseconds 

---

##### Questions <!-- .element: class="fragment" data-fragment-index="1" -->

* How can we prove the algorithm performs adequatly? <!-- .element: class="fragment" data-fragment-index="1" -->
* How do we monitor the resource consumption? <!-- .element: class="fragment" data-fragment-index="2" -->
* How do we compare resource usage between runs? <!-- .element: class="fragment" data-fragment-index="3" -->
* If we rely on a third party, how can we check its evolution? <!-- .element: class="fragment" data-fragment-index="4" -->
* Optionally, how can we check such requirements from CI? <!-- .element: class="fragment" data-fragment-index="5" -->

---
##### Initial attempt

With pytest, our test might look like:

```python [1-7]
import pytest
from my_package import is_prime


@pytest.mark.parametrize('n', [2, 3, 997, 104743, 982451653])
def test_prime(n):
    assert is_prime(n)
```

- Basic <!-- .element: class="fragment" data-fragment-index="1" -->
- *timeit* to assess time is a working (but poor) method<!-- .element: class="fragment" data-fragment-index="2" -->
- memory usage is not an out of the box feature <!-- .element: class="fragment" data-fragment-index="5" -->

---
 ## Pytest-monitor
---

##### About 

* Pytest plugin 
* Few requirements needed
* Small overhead 
* Track resources consumed by any test suite
    * Memory
    * Compute time (cpu, user, wall)
    * CPU usage
* Historize the results

---
##### Let's add it!

- With conda:
```bash
(my_conda_env) bash $> conda install pytest-monitor -c https://conda.anaconda.org/conda-forge
```

- With pip
```bash
(my_venv) bash $> pip install pytest-monitor
``` 

---
##### Results

Let's run it...
```bash
(my_conda_env) bash $> pytest --tag algo=naive --db monitor.db

==================== test session starts ====================
platform linux -- Python 3.6.8, pytest-4.4.1, py-1.8.0, [...]
rootdir: /home/user/projects/ospoxp/pytest-monitor
plugins: monitor-1.6.2
collected 5 items
tests/test_primality.py .....                         [ 100%]

=================== 5 passed 20.13 seconds ==================
```
---

##### Fetch data
```sql
sqlite> SELECT ITEM_VARIANT, MEM_USAGE, TOTAL_TIME, USER_TIME, KERNEL_TIME, CPU_USAGE, ITEM_PATH
   ...> FROM TEST_METRICS
   ...> WHERE ITEM_VARIANT = 'test_prime[982451653]';

ITEM_VARIANT|MEM_USAGE|TOTAL_TIME|USER_TIME|KERNEL_TIME|CPU_USAGE|ITEM_PATH
test_prime[982451653]|12009.296875|434.698642253876|423.74214976|8.03175076|0.99327179464212|test_prime
test_prime[982451653]|949.5234375|238.032186746597|237.562678592|0.2699172|0.999161495941682|test_prime
test_prime[982451653]|0.55078125|68.6835398674011|68.148762272|0.08830408|0.993499555843175|test_prime
test_prime[982451653]|0.7109375|33.8781900405884|33.681455904|0.007393368|0.994411130926371|test_prime
test_prime[982451653]|0.6953125|0.205749034881592|0.01341024|0.003311496|0.0812724881534601|test_prime
```

- Great, mission acomplished <!-- .element: class="fragment" data-fragment-index="1" -->
- Requires to know the sql structure <!-- .element: class="fragment" data-fragment-index="2" -->
- Does not support parallelism <!-- .element: class="fragment" data-fragment-index="3" -->

--- ---
 ## Monitor-server-API
---
##### About

Leverage pytest-monitor with 2 building blocks:

 - API (Python)
   * dedicated to query and fetch your data
   * works seemlessly with a local pytest-monitor database
---
##### About

Leverage pytest-monitor with 2 building blocks:

 - Server (REST)
   * manage a dedicated storage (through REST API) to insert metrics
   * enable parallelism in your test session (xdist support)

---

##### Full integration

```bash
(my_conda_env) bash $> export URL=http://my.monitor.org/api/v1
(my_conda_env) bash $> pytest --remote $URL --tag algo=naive \
                       --db monitor.db

===================== test session starts ===================
platform linux -- Python 3.6.8, pytest-4.4.1, py-1.8.0, [...]
rootdir: /home/user/projects/ospoxp/pytest-monitor
plugins: monitor-1.6.2
collected 5 items
tests/test_primality.py .....                         [ 100%]

==================== 5 passed 20.13 seconds =================
```

---

##### Example, fetching data with the API


```python
from monitor_server_api import Monitor, Field

mon = Monitor('http://my-monitor-server.org/api/v1')
sessions = mon.list_sessions()
metrics = mon.list_metrics_from(sessions)
df = metrics.to_df(sessions,
                   keep=[Field.ITEM_VARIANT, Field.TOTAL_TIME,
                         Field.ITEM_START_TIME, Field.MEM_USAGE,
                         Field.SCM])

```
---
<style>.container{
    display: flex;
}
.col{
    flex: 1;
}
</style>
<div class="container">
  <div class="col">
    <img src='ptm_wall.png'>
  </div>
  <div class="col">
    <img src='ptm_mem.png'>
  </div>
</div>

---

##### conclusion

* Easy to fetch and plot data
* No need to worry about internal data model
* Enables parallelism
* Perfect companion to pytest-monitor :)

--- ---

# Use Cases

---

### Know your dependencies

 - Migrating a dependencies (e.g.: pandas) can lead to
   - behavioral change
   - performance degradation on your core features
 
 - Tracking your application’s resource footprint can
   - prevent unwanted resource consumption
   - help you validate your requirements's version

---

### Know your tests

 - Applying load to a system can harm the performances
 - Analyzing resources can 
    - provide comprehensive view of tests and their category
    - help determining problems
    - validate new dev

--- ---
### Questions?
--- ---
### Addendum
---
##### Data Model

```
                  ┌─────────────────┐
                  │                 │
        ┌─────────┤  Test Metrics   ├───────┐
        │         │                 │       │
        │         └─────────────────┘       │
        │                                   │
        │                                   │
┌───────▼─────────┐                ┌────────▼───────┐
│                 │                │                │
│  Sessions Info  │                │  Machine Info  │
│                 │                │                │
└─────────────────┘                └────────────────┘
```


                     
---
