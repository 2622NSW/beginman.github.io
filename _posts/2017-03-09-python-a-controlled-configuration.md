---
layout: post
title: "Python 可控配置类"
description: ""
category: "python"
tags: [python]
---

[python-kafka](https://github.com/dpkp/kafka-python) KafkaProducer类写的真好，一个可控的配置类。抽象如下：

```python
import copy
import uuid


class Simple(object):
    DEFAULT_CONFIG = {
        'host': 'localhost',
        'client_id': None,
        'api_version': '1.0.0'
    }

    def __init__(self, **configs):
        self.config = copy.copy(self.DEFAULT_CONFIG)
        for key in self.config:
            if key in configs:
                self.config[key] = configs.pop(key)
        
        # Only check for extra config keys in top-level class
        assert not configs, 'Unrecognized configs: %s' % configs

        if self.config['client_id'] is None:
            self.config['client_id'] = str(uuid.uuid4())

if __name__ == '__main__':
    sim = Simple()
    print(sim.config)
    
    sim = Simple(host='192.168.1.129', client_id='demo')
    print(sim.config)

    # sim = Simple(api_version="1.1.1", other="other value")
    # AssertionError: Unrecognized configs: {'other': 'other value'} 

```

用起来很爽。
