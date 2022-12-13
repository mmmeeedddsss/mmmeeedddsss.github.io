---
title: TIL - Isort groups imports based on their sources
description: Adding a dependency to python was causing isort to behave differently.
date: 2022-12-13
categories: [CS, Python]
tags: [python, isort]     # TAG names should always be lowercase
---

_So I opened a PR and saw that one of the tests were failing.
When I check the logs, I saw that **isort is complaining** about a file that I haven't touched.
As we all do, I rebased my branch to master, and checked again.
I was still failing. I went and check the others' PR's and all was fine, except mine..._

```
ERROR: /home/jenkins/pipeline/../something_script.py Imports are incorrectly sorted.
--- /home/jenkins/pipeline/../something_script.py:before	2022-12-13 02:08:53
+++ /home/jenkins/pipeline/../something_script.py:after         2022-12-13 02:14:32
@@ -20,11 +20,10 @@
 import io
 import os

-from kafka import KafkaProducer
-
 import avro.io
 import avro.schema
 from avro.io import BinaryEncoder
+from kafka import KafkaProducer
```
---
I was adding a new dependency,`acryl-datahub==0.9.3.2`, to the project and that was the main suspect.
I removed the dependency from the requirements.txt(not pyproject.toml 😔), and isort was happy again.

Then I tried to remove the dependencies installed by `pip acryl-datahub==0.9.3.2` since this one by itself seems so unrelated. pip uninstall does not uninstall
the transitive dependencies installed with a dependency addition so manually found the list and tried to search for the one(s) that results in this complaining in an efficient manner,
by manually deleting them in bulks and see if isort keeps complaining xd.

At some point, I saw removing `avro==1.10.2` from the python environment was solving the issue.
When I check the error again with this in mind, I realized that the line `from kafka import KafkaProducer` was standing by itself initially,
and with the addition of the `avro==1.10.2`, it is grouped together with the `avro` imports.

I first thought that as a bug of isort, considering that we are using a very very old version of isort, 4.2.8, but nothing appeared online
when I search like: "Adding a dependency causes isort to behave differently", nothing related appeared.

Then I decide to go and read some documentation of isort and learn how it decides to group the imports and saw the following on its main page:

...

_(Code snipped with ugly looking mixed imports, and followed by:)_

After isort:
```python
from __future__ import absolute_import

import os
import sys

from third_party import (lib1, lib2, lib3, lib4, lib5, lib6, lib7, lib8,
                         lib9, lib10, lib11, lib12, lib13, lib14, lib15)

from my_lib import Object, Object2, Object3

print("Hey")
print("yo")
```

And I got the reason 😀🎉

---

isort groups third_party modules and my_lib kind of local modules separately.
When I go and add the dependency `avro==1.10.2` to the pyenv, now it can resolve
imports from `avro` and group these together with other third_party module, `kafka`.

I still wonder how we ended up with a code in our repository that does not have all its dependencies added to the requirements.txt but I guess why not.
I don't know how and where they use this file importing the non-existing module works.
It is named as `something_script.py`, maybe somebody just wanted to commit their ad-hoc script to the repository.

---

_This story is also the story that made me open this blog, together with some other recent such small discoveries. I hope I can share them regularly._
