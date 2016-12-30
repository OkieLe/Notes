## init服务

#### 情景:  定义一个init 启动的service, demo_service, 对应的执行档是/system/bin/demo

1. 创建一个demo.te 在`/device/mediatke/common/sepolicy`目录下
2. 定义demo 类型，init 启动service 时类型转换， demo.te 中
```
type  demo, domain;
type  demo_exec, exec_type, file_type;
init_daemon_domain(demo)
```
3. 绑定执行档 file_contexts 类型
```
/system/bin/demo  u:object_r:demo_exec:s0
```
4. 根据demo 需要访问的文件以及设备,  定义其它的权限在demo.te 中.


## 系统属性

#### 情景:  一个native service demo, 需要设置一个自定义的system property: demo.setting.case1

1. 定义system property类型. 在property.te
```
type demo_prop, property_type;
```
2. 绑定system property 类型.  在property_contexts
```
demo.   u:object_r:demo_prop:s0
```
3. 在demo.te 中新增
```
unix_socket_connect(demo,property,init);
allow demo demo_prop:property_service set;
```

## Binder服务

#### 情景:  一个native service demo, 创建了一个binder service demo, 并对外提供服务.

1. 在service.te 中定义service 类型
```
type  demo_service  service_manager_type;
```
2. 在service_contexts 中绑定service
```
demo  u:object_r:demo_service:s0
```
3. 在demo 中声明binder 权限  demo.te
```
binder_use(demo)
binder_call(demo, binderservicedomain)
biner_service(demo)
```
4. 允许demo 这个进程添加demo 这个service, 在demo.te
```
allow  demo demo_service:service_manager add;
```

## 访问系统设备

#### 情景:  一个native service 需要访问一个专属的char device /dev/demo

1. 定义device 类型, device.te
```
type demo_device dev_type;
```
2. 绑定demo device, file_contexts
```
/dev/demo u:object_r:demo_device:s0
```
3. 声明demo 使用demo device 的权限 demo.te
```
allow demo demo_device:chr_file rw_file_perms;
```

## 使用socket

#### 情景:  一个native service 通过init 创建一个socket 并绑定在 /dev/socket/demo, 并且允许某些process 访问.

1. 定义socket 的类型 file.te
```
type demo_socket, file_type;
```
2. 绑定socket 的类型 file_contexts
```
/dev/socket/demo_socket u:object_r:demo_socket:s0
```
3. 允许所有的process 访问.
```
# allow app connectto & write
unix_socket_connect(appdomain, demo, demo)
# more..
```