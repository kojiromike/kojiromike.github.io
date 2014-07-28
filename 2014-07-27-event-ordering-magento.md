---
layout: post
title: "Ways to Maintain Ordering of Event Observers in Magento 1"
description: "Michael A. Smith&#39;s opinion on something about Magento event observer ordering (with a handful of facts thrown in)"
category: 
tags: [php, magento]
---
{% include JB/setup %}

## Introduction
Suppose you want to inject some additional code after Magento saves the order, and it depends on `Mage_Tax_Model_Observer::salesEventOrderAfterSave`. Magento core already observes the "sales_order_save_after" event with that method:

'''
<config><global><events>
	<sales_order_save_after>
		<observers>
			<tax>
				<class>tax/observer</class>
				<method>salesEventOrderAfterSave</method>
			</tax>
		</observers>
	</sales_order_save_after>
</events></global></config>
```

Since event handler ordering is not specified, so how can we be sure our code immediately follows it?

## Rewrite and Inherit

The most obvious way to guarantee that an event is handled in the order you desire is to rewrite the existing class and override its observer:

```
<config><global><models><tax><rewrite>
	<observer>My_Module_Model_Observer</observer>
</rewrite></models></tax></global></config>
```

Here, we don't actually have to register the event observer because the one Magento has already registered will call our method. We just need to make sure our method actually invokes its parent

```
class My_Module_Model_Observer extends Mage_Tax_Model_Observer
{
	public function salesEventOrderAfterSave(Varien_Event_Observer $observer)
	{
		parent::salesEventOrderAfterSave($observer);
		// my code
	}
}
```

This approach is straightforward, but any time you rewrite a class in Magento you endanger upgradability and extensibility. Let's find a way that doesn't.

## Brute Force

Rather than rewriting the other observer, we can tell Magento not to call it as an observer and instead apply it directly ourselves:

```
<config><global><events>
	<sales_order_save_after>
		<observers>
			<my_module>
				<class>my/observer</class>
				<method>salesOrderSaveAfter</method>
			</my_module>
			<tax>
				<type>disabled>
			</tax>
		</observers>
	</sales_order_save_after>
</events></global></config>
```

and

```
class My_Module_Model_Observer
{
	public function salesEventOrderAfterSave(Varien_Event_Observer $observer)
	{
		Mage::getModel('tax/observer')->salesEventOrderAfterSave($observer);
		// my code
	}
}
```

Still pretty straightforward, and we avoided the override, but isn't there a way to just ensure the ordering of events?

## Well, Not Exactly...

Please?

## OK, but You're Not Going to Like It

The last-ditch, but possibly cleanest, way to guarantee event ordering is to realize that events are called in module order. In `Mage_Core_Model_App::dispatchEvent` when it loops over `$events[$eventName]['observers']` that array is ordered how the configs were loaded, which is the same as the order the modules were loaded. So if your observer really needs to come after 'tax/observer', just make sure your module is loaded after app/etc/Mage_All.xml (where Mage_Tax is loaded). Sadly, the only way to accomplish this is to name your module loader xml something that comes lexically after "Mage". It's ham-fisted, but it will work. I recommend commenting that file why it's named that way, particularly if it doesn't actually match your module name.


