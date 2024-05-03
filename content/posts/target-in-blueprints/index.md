+++
title = 'Home Assistant: using target in blueprints'
date = 2024-05-03
categories = ['blog']
tags = ['homeassistant', 'hass']
+++

{{< tldr >}}
* Here's [a snippet](#solution) for use in Home Assistant for turning an arbitrary `target` selector into a list of entity ids
* Also [here's](https://github.com/emoses/homeassistant-blueprints) a low-battery warning Blueprint that uses it.
{{< /tldr >}}


I've been using [tykeal's Low Battery
blueprint](https://community.home-assistant.io/t/low-battery-level-detection-notification-for-all-battery-sensors/258664/102)
 ([github](https://github.com/tykeal/homeassistant-blueprints/blob/main/low-battery.yaml)) in my Home Assistant setup for
some time, to give me an alert on any sensor batteries that need changing.  There are a number of sensors marked
"battery" that I don't care to track (for example iPhone batteries), and the blueprint has a spot where you can list
entities you want to exclude.

With the very exciting introduction of Labels [in HA
2024.4](https://www.home-assistant.io/blog/2024/04/03/release-20244/#labels-tag-everything-any-way-you-want), I decided
that, instead of listing excluded entities in the automation, I'd just tag them with a `NoBatteryWarning` label, and then
choose the label in the Blueprint input.  But it turns out that didn't work: the blueprint specified you had to list
entities directly.  I set out to fix it and it turned out it was pretty easy.

## Why doesn't it work?

Here's an excerpt of the original blueprint:

```yaml
blueprint:
  input:
    exclude:
      name: Excluded Sensors
      description: Battery sensors (e.g. smartphone) to exclude from detection.
        Only entities are supported, devices must be expanded!
      default: { entity_id: [] }
      selector:
        target:
          entity:
            device_class: battery
# ...

   sensors: >-
    {%- set result = namespace(sensors=[]) -%}
    {%- for state in states.sensor | selectattr('attributes.device_class', '==', 'battery') -%}
      {%- if 0 <= state.state | int(-1) < threshold | int and not state.entity_id in exclude.entity_id -%}
```

A `selector` is a thing which shows up in the UI and lets you pick from a filtered list.  The `target` selector is the
control you're used to seeing all over HA Settings pages, that looks like this: {{< figure
alt="Target selector image from HA doc, showing a chooser for entities, devices, areas, and labels" src="selector-target.png" >}}

It turns out when it's evaluated and you consume it a as a variable, you get a map that looks like this (I wish this were in [the doc](https://www.home-assistant.io/docs/blueprint/selectors/#target-selector) but I can't find it):

```json
{
    entity_id: [],
    device_id: [],
    area_id: [],
    label_id: [],
}
```

The original blueprint was only iterating over the `entity_id` list, so it was ignoring any other selections that you
made.

## The fix: just get all the entity ids {#solution}

But this isn't really hard to fix.  There are a set of functions that will give you all the entities in a label, area,
or device, called unsurprisingly `label_entities`, `area_entities` and `device_entities`.  I based the code on [This
forum
post](https://community.home-assistant.io/t/wth-howto-reference-a-device-areas-state-on-off-unavailable-with-a-target-selector/484428/43):

```yaml
  exclude: !input "exclude"
  exclude_entities: >
    {%- set ns = namespace(ret=[]) %}
    {%- for key in ['device', 'area', 'entity', 'label'] %}
      {%- set items = exclude.get(key ~ '_id', [])  %}
      {%- if items %}
        {%- set items = [ items ] if items is string else items %}
        {%- set items = items if key == 'entity' else items | map(key ~ '_entities') | sum(start=[]) %}
        {%- set ns.ret = ns.ret + [ items ] %}
      {%- endif %}
    {%- endfor %}
    {{ ns.ret | sum(start=[]) }}
# The same snippet from above but now it refers to exclude_entities instead of exclude.entity_id
  sensors: >-
    {%- set result = namespace(sensors=[]) -%}
    {%- for state in states.sensor | selectattr('attributes.device_class', '==', 'battery') -%}
      {%- if 0 <= state.state | int(-1) < threshold | int and not state.entity_id in exclude_entities -%}
```

So for each of the selector types (device, area, entity, and label), we expand them into entity ids and collect them
into a list, then we have a list of all the entities.

I forked tykeal's blueprint to my own repo, you can use it by clicking this badge:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Femoses%2Fhomeassistant-blueprints%2Fmaster%2Flow-battery.yaml)
