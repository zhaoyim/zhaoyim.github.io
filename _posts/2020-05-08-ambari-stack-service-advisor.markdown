---
layout:     post
title:      "Ambari stack service advisor"
subtitle:   "Ambari"
date:       2020-05-08 12:00:00
author:     "zhaoyim"
header-img: ""
catalog: true
tags:
    - Ambari
---

Ambari 主机选择和建议可通过service advisor中的colocateService接口来实现
![img](/img/in-post/ambari-host-collocate.png)

paste some code here:

```$xslt
  def colocateService(self, hostsComponentsMap, serviceComponents):
  #  # colocate HAWQSEGMENT with DATANODE, if no hosts have been allocated for HAWQSEGMENT
    self.logger.info("zhaoyim0000000 =====> %s" % (hostsComponentsMap))
    self.logger.info("zhaoyim1111111 =====> %s" % (serviceComponents))
    hawqSegment = [component for component in serviceComponents if component["StackServiceComponents"]["component_name"] == "TEST_MASTER"][0]
    if not self.isComponentHostsPopulated(hawqSegment):
      for hostName in hostsComponentsMap.keys():
        hostComponents = hostsComponentsMap[hostName]
        if {"name": "DATANODE"} in hostComponents and {"name": "TEST_MASTER"} not in hostComponents:
          hostsComponentsMap[hostName].append( { "name": "TEST_MASTER" } )
        if {"name": "DATANODE"} not in hostComponents and {"name": "TEST_MASTER"} in hostComponents:
          hostComponents.remove({"name": "TEST_MASTER"})
```


Ambari 参数建议和设定可通过service advisor中的getServiceConfigurationRecommendations接口来实现
![img](/img/in-post/ambari-configuration-recommendation.png)
paste some code here:

```$xslt
  def getServiceConfigurationRecommendations(self, configurations, clusterData, services, hosts):
    self.logger.info("Class: %s, Method: %s. TEST Recommending Service Configurations." %
                (self.__class__.__name__, inspect.stack()[0][3]))
    putTestEnvProperty = self.putProperty(configurations, "test-env", services)
    putTestEnvProperty('app_name', "zhaoyim")
```

service_advisor.py whole code here:

```$xslt
#!/usr/bin/env ambari-python-wrap
"""
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
"""
import imp
import os
import traceback
import re
import socket
import fnmatch
#import xml.etree.ElementTree as ET
from ambari_commons.str_utils import string_set_equals
from resource_management.core.logger import Logger

SCRIPT_DIR = os.path.dirname(os.path.abspath(__file__))
STACKS_DIR = os.path.join(SCRIPT_DIR, "../../../../")
PARENT_FILE = os.path.join(STACKS_DIR, "service_advisor.py")

try:
  if "BASE_SERVICE_ADVISOR" in os.environ:
    PARENT_FILE = os.environ["BASE_SERVICE_ADVISOR"]
  with open(PARENT_FILE, "rb") as fp:
    service_advisor = imp.load_module("service_advisor", fp, PARENT_FILE, (".py", "rb", imp.PY_SOURCE))
except Exception as e:
  traceback.print_exc()
  print "Failed to load parent"


class TestServiceAdvisor(service_advisor.ServiceAdvisor):

  def __init__(self, *args, **kwargs):
    self.as_super = super(TestServiceAdvisor, self)
    self.as_super.__init__(*args, **kwargs)

    self.initialize_logger("TestServiceAdvisor")

    # Always call these methods
    self.modifyMastersWithMultipleInstances()
    self.modifyCardinalitiesDict()
    self.modifyHeapSizeProperties()
    self.modifyNotValuableComponents()
    self.modifyComponentsNotPreferableOnServer()
    self.modifyComponentLayoutSchemes()

  def getServiceConfigurationRecommendations(self, configurations, clusterData, services, hosts):
    self.logger.info("Class: %s, Method: %s. TEST Recommending Service Configurations." %
                (self.__class__.__name__, inspect.stack()[0][3]))
    putTestEnvProperty = self.putProperty(configurations, "test-env", services)
    putTestEnvProperty('app_name', "zhaoyim")

    componentsListList = [service["components"] for service in services["services"]]
    componentsList = [item["StackServiceComponents"] for sublist in componentsListList for item in sublist]
    #testHosts = self.getHosts(componentsList, "TEST")
    testHosts = self.getHostsWithComponent("TEST", "TEST_MASTER", services, hosts)
    for h in testHosts:
      self.logger.info("zhaoyim1 =====> 11111111 %s" % (h["Hosts"]["host_name"]))

    self.logger.info("zhaoyim1 =====> 222222222 %s" % (testHosts[0]["Hosts"]["host_name"])) 

    nameHosts = self.getHosts(componentsList, "NAMENODE")
    self.logger.info("zhaoyim2 =====> %s" % (nameHosts))

    livyHosts = self.getHosts(componentsList, "LIVY2_SERVER")
    self.logger.info("zhaoyim3 =====> %s" % (livyHosts))

    #hiveHosts = self.getHosts(componentsList, "HIVE_CLIENT")
    hiveHosts = self.getHostsWithComponent("HIVE", "HIVE_CLIENT", services, hosts)
    self.logger.info("zhaoyim4 =====> %s" % (hiveHosts))
    
    #sparkHosts = self.getHosts(componentsList, "SPARK2_THRIFTSERVER")
    #phoenixHosts = self.getHosts(componentsList, "PHOENIX_QUERY_SERVER")
    nfsHosts = self.getHosts(componentsList, "NFS_GATEWAY")
    self.logger.info("zhaoyim5 =====> %s" % (nfsHosts))

    #if "test-env" in services["configurations"]:
    #  if 'forced-configurations' not in services:
    #    services["forced-configurations"] = []

    #    old_properties = services["configurations"]["test-env"]["properties"]
    #    putTestEnvProperty = self.putProperty(configurations, "test-env", services)
    #    putTestEnvProperty('app_name', "zhaoyim")

        #new_properties = {}
        #tree = ET.parse(os.path.join(SCRIPT_DIR, 'configuration/test-env.xml'))
        #for el in tree.findall('.//property'):
        #  new_properties[el.find('.//name').text]=el.find('.//value').text

        # Add new properties
        #for property, value in new_properties.iteritems():
        #  if property not in old_properties:
        #    services["forced-configurations"].append({"type": "test-env", "name": property})
        #    putTestEnvProperty(property, value)
  def colocateService(self, hostsComponentsMap, serviceComponents):
  #  # colocate HAWQSEGMENT with DATANODE, if no hosts have been allocated for HAWQSEGMENT
    self.logger.info("zhaoyim0000000 =====> %s" % (hostsComponentsMap))
    self.logger.info("zhaoyim1111111 =====> %s" % (serviceComponents))
    hawqSegment = [component for component in serviceComponents if component["StackServiceComponents"]["component_name"] == "TEST_MASTER"][0]
    if not self.isComponentHostsPopulated(hawqSegment):
      for hostName in hostsComponentsMap.keys():
        hostComponents = hostsComponentsMap[hostName]
        if {"name": "DATANODE"} in hostComponents and {"name": "TEST_MASTER"} not in hostComponents:
          hostsComponentsMap[hostName].append( { "name": "TEST_MASTER" } )
        if {"name": "DATANODE"} not in hostComponents and {"name": "TEST_MASTER"} in hostComponents:
          hostComponents.remove({"name": "TEST_MASTER"})

  def getServiceComponentLayoutValidations(self, services, hosts):
    componentsListList = [service["components"] for service in services["services"]]
    componentsList = [item["StackServiceComponents"] for sublist in componentsListList for item in sublist]

    testMasterHosts = self.getHosts(componentsList, "TEST_MASTER")
    self.logger.info("zhaoyim ++++++++++ =====> 111 %s" % (testMasterHosts))

    metronParsersHost = self.getHostsWithComponent("TEST", "TEST_MASTER", services, hosts)[0]["Hosts"]["host_name"]
    self.logger.info("zhaoyim ++++++++++ =====> 222 %s" % (metronParsersHost))

    return []

  def modifyMastersWithMultipleInstances(self):
    """
    Modify the set of masters with multiple instances.
    Must be overriden in child class.
    """
    # Nothing to do
    pass

  def modifyCardinalitiesDict(self):
    """
    Modify the dictionary of cardinalities.
    Must be overriden in child class.
    """
    # Nothing to do
    pass
  def modifyHeapSizeProperties(self):
    """
    Modify the dictionary of heap size properties.
    Must be overriden in child class.
    """
    pass

  def modifyNotValuableComponents(self):
    """
    Modify the set of components whose host assignment is based on other services.
    Must be overriden in child class.
    """
    # Nothing to do
    pass

  def modifyComponentsNotPreferableOnServer(self):
    """
    Modify the set of components that are not preferable on the server.
    Must be overriden in child class.
    """
    # Nothing to do
    pass
  def modifyComponentLayoutSchemes(self):
    """
    Modify layout scheme dictionaries for components.
    The scheme dictionary basically maps the number of hosts to
    host index where component should exist.
    Must be overriden in child class.
    """

    pass  
```