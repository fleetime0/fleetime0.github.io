---
title: vsomeip-sd模块源码分析
date: 2024-04-17 14:57 +0800
last_modified_at: 2024-04-19 11:21 +0800
author: AClumsyDog
categories: ["源码分析", "vsomeip"]
tags: ["c++", "someip", "autosar"]
pin: true
math: true
mermaid: true
img_path: /assets/img/vsomeip-sd-code-analysis/
---

## 服务发现UML类图

服务发现模块类之间的关系较为复杂，为了方便理解。这里先对各个主要类进行简单的说明。

**注意**：类的详细说明见链接。

- **runtime和runtime_impl**：

  service discovery类的构造工厂类。可以创建service_discovery_impl类的实例。runtime是纯虚类，runtime_impl继承runtime。

- **service_discovery_host**：

  纯虚类，Routing Manager相关类（没有包括在此UML类图中）继承于此类。service_discovery_impl类中包括指向此类的指针，所以service_discovery_impl可以通过指向service_discovery_host的指针间接使用Routing Manager相关类（例如：发送消息）。

- **service_discovery和service_discovery_impl**：

  service_discovery是纯虚类，service_discovery_impl继承service_discovery类实现所有的纯虚函数。服务发现相关的处理逻辑在此类中实现。

- **request**：

  保存请求服务相关的信息。service_discovery_impl中保存有request的容器，可以处理相关服务发现的逻辑。

- **subscription**：

  pass

- **remote_subscription_ack**：

  pass

- **message_element_impl**：

  所有message相关类的基类，message_elemen_impl中有指向message_impl的指针。其他类（entry_impl相关类和option_impl相关类）可以通过调用`get_owning_message()`获得指向message_impl类指针。

- **message_impl**：

  组合所有message相关的类，`entries_`（`std::vector<std::shared_ptr<entry_impl>>`）和`options_`（`std::vector<std::shared_ptr<option_impl>>`）保存了所有的entry_impl相关类和option_impl相关类。

- **entry_impl**：

  entry相关类的基类。

- **option_impl**：

  option相关类的基类。

```mermaid
classDiagram
    class message_impl {
        -entries_t entries_
        -options_t options_
        +get_entries() const entries_t&
        +get_options() const options_t&
        +add_entry_data(const std::shared_ptr~entry_impl~&, const std::vector~std::shared_ptr~option_impl~~&, const std::shared_ptr~entry_impl~&) bool
        +find_option(const std::shared_ptr~option_impl~&) std::shared_ptr~option_impl~
        +get_option_index(const std::shared_ptr~option_impl~&) int16_t
        +get_option(int16_t) std::shared_ptr~option_impl~
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
        -deserialize_entry(vsomeip_v3::deserializer *_from) entry_impl*
        -deserialize_option(vsomeip_v3::deserializer *_from) option_impl*
    }

    message_impl o-- option_impl : Aggregation
    message_impl o-- entry_impl : Aggregation

    class message_element_impl {
        -message_impl* owner_
        +message_element_impl()
        +get_owning_message() message_impl*
        +set_owning_message(message_impl* _owner)
    }
    message_element_impl --> message_impl : Association

    class option_impl {
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
    }
    message_element_impl <|-- option_impl : Inheritance

    class configuration_option_impl {
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
    }
    option_impl <|-- configuration_option_impl : Inheritance

    class selective_option_impl {
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
    }
    option_impl <|-- selective_option_impl : Inheritance

    class load_balancing_option_impl {
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
    }
    option_impl <|-- load_balancing_option_impl : Inheritance

    class protection_option_impl {
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
    }
    option_impl <|-- protection_option_impl : Inheritance

    class ip_option_impl {
    	<<Abstract>>
        +serialize(vsomeip_v3::serializer*) bool*
        +deserialize(vsomeip_v3::deserializer*) bool*
    }
    option_impl <|-- ip_option_impl : Inheritance

    class ipv4_option_impl {
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
    }
    ip_option_impl <|-- ipv4_option_impl : Inheritance

    class ipv6_option_impl {
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
    }
    ip_option_impl <|-- ipv6_option_impl : Inheritance

    class entry_impl {
        +assign_option(const std::shared_ptr~option_impl~&)
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
    }
    message_element_impl <|-- entry_impl : Inheritance
    entry_impl ..> option_impl : Dependency

    class eventgroupentry_impl {
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
        +matches(const eventgroupentry_impl&, const message_impl::options_t&) bool
        +get_selective_option() std::shared_ptr~selective_option_impl~
    }
    entry_impl <|-- eventgroupentry_impl : Inheritance
	eventgroupentry_impl ..> selective_option_impl : Dependency
	eventgroupentry_impl ..> option_impl : Dependency

    class serviceentry_impl {
        +serialize(vsomeip_v3::serializer*) bool
        +deserialize(vsomeip_v3::deserializer*) bool
    }
    entry_impl <|-- serviceentry_impl : Inheritance

	class deserializer {
		+deserialize_sd_message message_impl*
	}
	deserializer ..> message_impl : Dependency

    class remote_subscription_ack {
        -std::vector~std::shared_ptr~message_impl~~ messages_
        +get_messages() std::vector~std::shared_ptr~message_impl~~
        +get_current_message() std::shared_ptr~message_impl~
        +add_message() std::shared_ptr~message_impl~
    }
    remote_subscription_ack o-- message_impl : Aggregation

    class request

    class runtime {
    	<<Abstract>>
    	+create_service_discovery(service_discovery_host*, std::shared_ptr~configuration~) std::shared_ptr~service_discovery~*
    }

    class runtime_impl {
    	+create_service_discovery(service_discovery_host*, std::shared_ptr~configuration~) std::shared_ptr~service_discovery~*
    }
    runtime <|-- runtime_impl : Inheritance
    runtime_impl ..> service_discovery_host : Dependency
    runtime_impl ..> service_discovery : Dependency

    class subscription

	class service_discovery {
		<<Abstract>>
	}

	class service_discovery_host {
		<<Abstract>>
	}

	class service_discovery_impl {
		-service_discovery_host *host_
		-std::shared_ptr~deserializer~ deserializer_
		-requests_t requested_
		-std::map~service_t, std::map~instance_t, std::map~eventgroup_t, std::shared_ptr~subscription~~~~ subscribed_
		-std::map~std::shared_ptr~remote_subscription~, std::shared_ptr~remote_subscription_ack~~ pending_remote_subscriptions_
		-process_serviceentry(std::shared_ptr~serviceentry_impl~&, const std::vector~std::shared_ptr~option_impl~~&, bool, std::vector~std::shared_ptr~message_impl~~&, bool, const sd_acceptance_state_t&)
		-process_offerservice_serviceentry(service_t, instance_t, major_version_t, minor_version_t, ttl_t, const boost::asio::ip::address&, uint16_t, const boost::asio::ip::address&, uint16_t, std::vector~std::shared_ptr~message_impl~~&, bool, const sd_acceptance_state_t&)
		-send(const std::vector~std::shared_ptr~message_impl~~&) bool
		-serialize_and_send(const std::vector~std::shared_ptr~message_impl~~&, const boost::asio::ip::address&)
		-create_subscription(major_version_t, ttl_t, const std::shared_ptr~endpoint~&, const std::shared_ptr~endpoint~&, const std::shared_ptr~eventgroupinfo~&) std::shared_ptr~subscription~
		-send_subscription(const std::shared_ptr~subscription~&, const service_t, const instance_t, const eventgroup_t, const client_t)
		-insert_find_entries(std::vector~std::shared_ptr~message_impl~~&, const requests_t &)
		-find_request(service_t, instance_t) std::shared_ptr~request~
		-insert_subscription_ack(const std::shared_ptr~remote_subscription_ack~&, const std::shared_ptr~eventgroupinfo~&, ttl_t, const std::shared_ptr~endpoint_definition~&, const std::set~client_t~&)
		-send_subscription_ack(const std::shared_ptr~remote_subscription_ack~&)
	}
	service_discovery <|-- service_discovery_impl : Inheritance
	service_discovery_impl --> service_discovery_host : Association
	service_discovery_impl o-- deserializer : Aggregation
	service_discovery_impl ..> message_impl : Dependency
	service_discovery_impl o-- subscription : Aggregation
	service_discovery_impl o-- request : Aggregation
	service_discovery_impl o-- remote_subscription_ack : Aggregation
```

## offer service流程

```mermaid
flowchart
    subgraph main阶段
        direction TB
        subgraph graph1
            direction TB
            graph1-A(start_main_phase_timer) --> graph1-B(延时cyclic_offer_delay_，异步执行on_main_phase_timer_expired)
        end
        subgraph graph2
            direction TB
            graph2-A(on_main_phase_timer_expired) -->graph2-B{判断服务是否进入main阶段}
            graph2-B -->|yes| graph2-C(发送报文)
            graph2-B -->|no| graph2-D(调用start_main_phase_time)
            graph2-C --> graph2-D
        end
    end
```

```mermaid
flowchart TD
    subgraph initial和repet阶段
        direction LR
        subgraph graph1
            direction TB
            graph1-A(offer_service) --> graph1-B{判断是否在collected_offers中找到}
            graph1-B -->|yes| C(none)
            graph1-B -->|No| graph1-D(将offer的服务添加入collected_offers_)
        end
        subgraph graph2
            direction TB
            graph2-A(start_offer_debounce_timer) --> graph2-B{判断是否为首次运行}
            graph2-B -->|yes| graph2-C(延时设置为initial_delay_)
            graph2-B -->|No| graph2-D(延时设置为offer_debounce_time_)
            graph2-D --> graph2-E(异步执行on_offer_debounce_timer_expired)
            graph2-C --> graph2-D
        end
        subgraph graph3
            direction TB
            graph3-A(on_offer_debounce_timer_expired) --> graph3-B{判断是否为诊断模式}
            graph3-B -->|yes| graph3-C[大致就是将collected_offers_变量中的非someip服务导入repetition_phase_offers]
            graph3-B -->|No| graph3-D[collected_offers_导入repetition_phase_offers]
            graph3-D --> graph3-E["发送第一个offer(初始化阶段结束)"]
            graph3-E --> graph3-F[重复阶段时间变量赋值，将重复阶段的offer和times信息存入repetition_phase_timers_变量，异步执行on_repetition_phase_timer_expired]
        end
    end
```

![VSOMEIP-SD-Offer-Repet](VSOMEIP-SD-Offer-Repet.png)

### app的offer_service 梳理

#### routing_manager_impl 路线

几个关键的容器：

1. routing_manager_impl 的私有 map 型容器 offer_commands\_，pending_offers\_
2. routing_manager_base 的 protected 的 map 型容器 local_services_，（由于是 protected 的，所以该容器应该是 routing_manager_impl 和 routing_manager_proxy 共享的）
3. routing_manager_base 的私有的 services_，该容器会记录所有已经注册了的服务的信息，包括 local 和 remote 的服务，在服务发现的main阶段，sd模块会调用routing_manager_base提供的公共接口routing_manager_base::get_services来获取已经注册了的服务的信息，然后发送offer_service报文

```mermaid
flowchart
A("offer_service(_client, _service, _instance, _major, _minor, true)")-->B("insert_offer_command(_service, _instance, VSOMEIP_OFFER_SERVICE, _client, _major, _minor)")
B-->C("通过 uid 和 gid 来检查本次的 offer_service 命令是否有权限")
C-->|no|D("erase_offer_command(_service, _instance)")
C-->|yes|E("handle_local_offer_service(_client, _service, _instance, _major, _minor)")
D-->Q("返回false")
E-->|false|Z("返回false")
E-->|true|F("init_service_info(_service, _instance, true)")
F-->G("Service Discovery是否使能")
G-->|使能|H("通过 SD 实例来发送 offer service entry")
G-->|没使能|I("send_pending_subscriptions(_service, _instance, _major)")
H-->I
I-->J("stub->on_offer_service(_client, _service, _instance, _major, _minor)")
J-->K(on_availability)
```

routing_manager_impl 路由管理器的实例中通过一个私有的map型容器 offer_commands_ 来维护所有来自app层的offer_service()调用，其具体类型定义如下：

```c++
std::map<std::pair<service_t, instance_t>, std::deque<std::tuple<uint8_t, client_t, major_version_t, minor_version_t>>> offer_commands_; 
```

insert_offer_command() 就是将当前来自 app 层的一个 offer_service 命令存入到这个容器中。

insert_offer_command()会从安全性考虑，结合 uid 和 gid 来检查本次的 offer_service 命令是否有权限，若没权限，就从 offer_commands_ 容器中将刚存入offer_service通过`erase_offer_command(_service, _instance)`删掉，并返回。

routing_manager_impl 继承的父类 routing_manaer_base 中使用一个 map 类型的容器 local_services_ 来维护本地服务，其具体类型定义如下：

```c++
typedef std::map<service_t, std::map<instance_t, std::tuple<major_version_t, minor_version_t, client_t>>> local_services_map_t;
local_service_map_t local_services_;
```

handle_local_offer_service()中会先进行以下几个方面的检查工作：

```mermaid
flowchart
A("从local_services_容器中查找当前上层 app想要注册的服务")
A-->|不存在当前app|B("从 pending_offers_ 容器中查看是否已经有其它 app 已经在排队等待注册当前 app 本次想要注册的服务")
A-->|存在|Q("返回false")
B-->|不存在|C("当前 app 是否也已经在排队等待注册本次想要注册的服务")
B-->|存在|L(返回false)
C-->|不存在|D("其它 app 注册了当前 app 本次想要注册的服务，当前 app 并不在排队等待中，确认此前注册服务的 app 的是否还处于 alive 状态")
C-->|存在|M(返回false)
D-->|alive|U(返回false)
D-->|no|E("当前 app 的信息填入到 pending_offers_，待时机到了再执行注册")
A-->|完全不存在服务|F("检查该服务是否已经被远端节点注册")
```



1. 从 local_services_ 容器中查找当前上层 app 想要注册的服务，若存在，则表示该上层 app 此前已经在本地注册过本次想要注册的服务了，函数直接返回 false，避免重复注册；否则进入2。
2. 从 pending_offers_ 容器中查看是否已经有其它 app 已经在排队等待注册当前 app 本次想要注册的服务了，若是的，则函数直接返
   回 false，避免重复注册； 否则进入3。
3. 在 2 中同时会检查当前 app 是否也已经在排队等待注册本次想要注册的服务（因为一个 app 可能会周期性地调用 offer_service来注册一个服务），若是当前 app 确实是已经在排队中，则函数直接返回 false，避免重复注册；否则进入4。
4. 如果此前已经有其它 app 注册了当前 app 本次想要注册的服务，且当前 app 并不在排队等待中，则再确认此前注册服务的 app 的是否还处于 alive 状态，若是 alive 状态，则函数直接返回 false，避免重复注册；否则将当前 app 的信息填入到 pending_offers_ 这个记录排队等待的 app 的容器中，等待时机到了再执行注册。
5. 若 local_services_ 容器中查找不到当前上层 app 本次想要注册的服务，则还会检查该服务是否已经被远端节点注册了（offered remotely），通过调用 routing_manager_base::offer_service(_client, _service, _instance, _major, _minor)来完成该项检查，就是从 services_ 中查找。若该服务已经被远端节点注册了，则拒绝本地的本次注册；否则就本次想要注册的服务的信息填充到 services_ 和 services_remote_中，完成本次服务注册。

init_service_info主要是根据json配置中的服务的端口，调用 find_or_create_server_endpoint()函数来创建和初始化服务端口（tcp/udp）,并在 services_ 中对服务端口信息做好记录。

如果使能了 Service Discovery，则调用：

```c++
std::share_ptr<serviceinfo> its_info = find_service(_service, _instance);
if (its_info) {
	discovery_ -> offer_service(its_info); // 通过 SD 实例来发送 offer service entry
}
```

send_pending_subscriptions函数流程如下：

```mermaid
flowchart
A("send_pending_subscriptions")-->B("routing_manager_impl::send_subscribe()")
B-->C("stub_->send_subscribe()")
```

on_availability函数流程如下：

```mermaid
flowchart
A("on_availability(_service, _instance, _true, _major, _minor)")-->B("host->on_availability(_service, _instance, _is_available, _major, _minor)")
```

最后触发 client 端的on_availability()回调函数。

#### routing_manager_proxy 路线

```mermaid
flowchart
A("routing_manager_base::offer_service(_client, _service, _instance, _major, _minor)")-->B("send_offer_service(_client, _service, _instance, _major, _minor)")
```

routing_manager_base::offer_service在 routing_manager_impl 的路线 handle_local_offer_service()中也出现了，就是检查本次想注册的服务是否已经被本地或远端的其它应用注册了，若没有，则完成此次注册；否则就放弃此次注册。

send_offer_service流程如下：

```mermaid
flowchart
A("send_offer_service(_client, _service, _instance, _major, _minor)")-->B("sender_ -> send(its_command, sizeof(its_command))")
B -->C("local_client_endpoint_impl::send()")
```

proxy这边的offer_service的逻辑，就是构建了一个VSOMEIP_OFFER_SERVICE的指令，通过unix域通信的方式，把指令发给了host路由，host路由的模块响应该指令的地方就是在routing_manager_stub中的on_message方法中。

从上可知，由于 SD 的实例只会在 routing_manager_impl 实例中创建和初始化一次，所以 routing_manager_proxy 路线的 offer_service() 只会通过 local_client_endpoint_impl 这个端口来传送，完成服务注册。而后续会按需由 routing_manager_impl 来通过 SD 实例接口将注册的服务发布给远端。

### 服务发现的offer_service 梳理

#### 多播报文的发送

**几个关键容器：**

1. service_discovery_impl的私有map型容器collected_offers_，collected_offers_是一个三维数组[srvice][instance][info]，该变量通过service_discovery_impl::offer_service函数从outing_manager_base 的私有的 services_容器中拷贝而来所需要的serviceinfo，sd的stop_offer_service与之相反，collected_offers_变量会由on_offer_debounce_timer_expired函数读取
2. rservice_discovery_impl的私有map型容器epetition_phase_timers_，该容器存放repetition_phase_timers_类型指针和services_t

**紧接上文的sd模块offer service讲解**
如果使能了 `Service Discovery`，则调用:

```c++
std::share_ptr<serviceinfo> its_info = find_service(_service, _instance);
//--->find_service为routing_manager_base::find_service函数
if (its_info) {
     discovery_ -> offer_service(its_info); // 通过 SD 实例来发送 offer service entry
 }
 offer_service(const std::shared_ptr<serviceinfo> &_info);
```
从 `collected_offers_ `容器中查找当前想要添加的服务是否存在，若不存在，则添加。该函数由 `routing_manager_impl `调用

具体调用顺序为：`app层start()--->routing_manager_impl::start()->on_net_interface_or_route_state_changed->start_ip_routing()--->service_discovery_impl::start()
--->service_discovery_impl::start()`.

该函数首先会检查sd的多播端口是否创建，之后会调用以下函数：

```c++
    start_main_phase_timer();
    start_offer_debounce_timer(true);
    start_find_debounce_timer(true);
    start_ttl_timer();
```
- `--->start_main_phase_timer();`
      该函数会异步延时`cyclic_offer_delay_`之后执行`on_main_phase_timer_expired`函数

  ```c++
  --->on_main_phase_timer_expired(const boost::system::error_code &_error);
       --->send(true);//该函数调用service_discovery_impl::send(bool _is_announcing)
            --->insert_offer_entries(its_messages, its_offers, false);
            --->return send(its_messages);
            --->start_main_phase_timer();
  ```
  
  在`send(true)`函数内会调用`insert_offer_entries`函数，该函数原型为:
  
  ```c++
  insert_offer_entries(std::vector<std::shared_ptr<message_impl> > &_messages,const services_t &_services, bool _ignore_phase);
  ```
  
  该函数具体代码为:
  
  ```c++
   if ((_ignore_phase || its_instance.second->is_in_mainphase())
   		&& (its_instance.second->get_endpoint(false)
   		|| its_instance.second->get_endpoint(true))) {
   	insert_offer_service(_messages, its_instance.second);
   }
  ```
  
  若第三个变量为`false`，则会判断对应的服务实例是否进入`main`阶段，若没有进入则不进行处理，直接返回，因此`its_messages`为空，然后再`send(its_messages); `      
  
  其中，由于 `its_messages `的`entry`字段为空，因此不会发送`offer service`报文。否则，代表进入`main`阶   段，开始循环`cyclic_offer_delay_`发送`offerservice`报文.

  最后再调用`start_main_phase_timer();`重启main_phase定时器，循环往复

- `--->start_offer_debounce_timer(true);`
      首先会判断是否第一次开始，若是首次开始，则初始化延时`initial_delay_`，否则延时`offer_debounce_time_`，该时间为去抖动时间，即两个发送报文最短时间间隔，然后会异步调用`on_offer_debounce_timer_expired`函数
     - `--->on_offer_debounce_timer_expired(const boost::system::error_code &_error);`
           该函数会依据`collected_offers_`内的服务发送第一个`offer service`报文，作为`initial`阶段结束，并将服务转移至 `repetition_phase_timers`_容器中，然后判断变量`repetitions_max_`是否为0，为0则延时`cyclic_offer_delay_`直接进入`main`阶段，否则延时`repetitions_base_delay_`，并设置`its_repetitions`为1，统计循环发送`offer service`报文次数，待延时结束后，则直接异步调用`on_repetition_phase_timer_expired`函数.
           该函数原型为：
           ```c++
           on_repetition_phase_timer_expired(const boost::system::error_code &_error,
                    			const std::shared_ptr<boost::asio::steady_timer>&_timer,
                               std::uint8_t _repetition, std::uint32_t _last_delay);
         ```
           该函数首先会判断重复次数是否剩余为0，为0则调用以下函数
        
        ```c++
        move_offers_into_main_phase(_timer);
        ```
       
        将`repetition_phase_timers_`容器中的服务`is_in_mainphase`属性设置为真，并将容器删除,然后会发送`repetition_phase_timers_`容器中保存的服务`offer service`报文，延时异步再次调用`on_repetition_phase_timer_expired`函数，之后当发送次数达到`repetitions_max_`最大值时，则执行`move_offers_into_main_phase`函数
       至此，`initial`和`repet`阶段结束，`offer service`的服务服务`is_in_mainphase`属性为真，`on_main_phase_timer_expired`函数中的`send`开始正常执行，发送`main`阶段的`offer service`报文
        
