---
title: "Backend Python:Working With Python Lxml Parser for Creating XML Elements"
date: 2018-09-07T21:44:59+03:00
draft: false 
categories: ["backend", "python"]
tags: ["python"]
---

lxml parser can be a bit confusing because of the sheer range of options it offers. Here are a few cookbook style examples.

## XML Generation
Target code:
```xml
<root_element xmlns="http://www.w3.org/TR/html4/" some_more_params="12345-678-ABC" yet_more_params="POKEMON-SUCKS">
  <element>
    <element_data1 type="sometype"><![CDATA[SomeVal123]]></element_data1>
    <element_data2>12345</element_data2>
    <element_data3 some_attr="some_attr">More random data</element_data3>
  </element>
  <another_element>
    <element_data1 type="sometype"><![CDATA[SomeVal1234]]></element_data1>
    <element_data2>12345</element_data2>
    <element_data3 some_attr="some_attr">More random data</element_data3>
  </another_element>
</root_element>
```
Ok, Here is the code to generate it:
```python
from lxml import etree

def create_xml():
    XML_OPTIONS = {'pretty_print': True, 'xml_declaration': True, 'encoding': 'utf-8'}

    def create_root_element():
        XHTML_NAMESPACE = "http://www.w3.org/TR/html4/"
        
        root_element = etree.Element('root_element', nsmap = {None: XHTML_NAMESPACE}, some_more_params="12345-678-ABC", yet_more_params="POKEMON-SUCKS")
        return root_element
    def create_element(name, data):
        element = etree.Element(name)
        for key, value in data.items():
            if key == 'element_data1':
                etree.SubElement(element, 'element_data1', type='sometype').text = etree.CDATA(value)
            elif key == 'element_data3':
                etree.SubElement(element, 'element_data3', some_attr='some_attr').text = str(value)
            else:
                etree.SubElement(element, key).text = str(value)
        return element

    all_data = {
        'element': {
            'element_data1': 'SomeVal123',
            'element_data2': 12345,
            'element_data1': 'More random data'
        },
        'another_element': {
            'element_data1': 'SomeVal1234',
            'element_data2': 12345,
            'element_data1': 'More random data'
        }
    }

    root = create_root_element()
    for key, value in all_data.items():
        element = create_element(name=key, data=value)
        root.append(element)
    return etree.tostring(root, **XML_OPTIONS)
    
print(create_xml())
```
Notice the `nsmap = {None: XHTML_NAMESPACE}` line in `etree.Element('root_element', nsmap = {None: XHTML_NAMESPACE}, some_more_params="12345-678-ABC", yet_more_params="POKEMON-SUCKS")`. This is required if you wish to provide a namespace.

Note:If you passed it like so `etree.Element('root_element', nsmap, some_more_params="12345-678-ABC", yet_more_params="POKEMON-SUCKS")` with `nsmap = {None: XHTML_NAMESPACE}`, i.e. nsmap as a variable, you will get an error.