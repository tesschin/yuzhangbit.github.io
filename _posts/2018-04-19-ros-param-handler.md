---
published: true
title: rosparam_handler usage
classes: wide
categories:
  - Tools
tags:
  - ROS
  - Programming
---

There are two ways to set parameters in ROS. One is loading parameter value from yaml files to the [parameter_server](http://wiki.ros.org/Parameter%20Server). It's not flexible to adjust the values of parameters on parameter_server on the fly.
The parameters are passed to your app classes by the private ros node handle. You have to create the variables in the class to hold these values.

The second method is using `dynamic_reconfigure`. You define types of parameters in a python file. Then the `dynamic_reconfigure` tool will generate a header file that you can call directly. It seems like an interface file generated by IDL (Interface Definition Language) such as rosmsg and protobuf. You can adjust the values of parameters through the dynamic_reconfigure gui in rqt on the fly.
Thus the parameter_server sounds like a static configuration tool and dynamic_reconfigure is a dynamic configuration tool for parameters.

The [rosparam_handler](https://github.com/cbandera/rosparam_handler) is a nice tool that unifies these two functionalities and makes these two types of parameters live in the same namespace to avoid redundancy.


The orignal tutorials did a great job in explaining the details in seperate tutorials. Here I want to present the necessary changes you need to do from a global perspective. Before diving into the details, please put the `rosparam_handler` package in your `catkin_ws/src` folder.
```
git clone https://github.com/cbandera/rosparam_handler
```
***Steps:***   
1. Define the parameter types and properties in the `cfg/Some.params` file. Note this file is used to generate header-only class that you can call. If you don't have the `cfg` folder, just create one using `mkdir -p cfg` in your package root folder.   

    ```python
    #!/usr/bin/env python
    from rosparam_handler.parameter_generator_catkin import *
    gen = ParameterGenerator()
    gen.add("int_param", paramtype="int", description="An Integer parameter")
    gen.add("double_param", paramtype="double",description="A double parameter")
    gen.add("bool_param", paramtype="bool", description="A Boolean parameter")
    gen.add("vector_param", paramtype="std::vector<double>", description="A vector parameter")
    gen.add("map_param", paramtype="std::map<std::string,std::string>", description="A map parameter")
    gen.add("configurable_parameter", paramtype="double", description="This parameter can be set via dynamic_reconfigure", configurable=True)
    gen.add("dummy", paramtype="double", description="My Dummy parameter", level=0,
    edit_method="", default=5.2, min=0, max=10, configurable=True,
    global_scope=False, constant=False)
    gen.add_enum("my_enum", description="My first self written enum",
    entry_strings=["Small", "Medium", "Large", "ExtraLarge"], default="Medium")
    exit(gen.generate("package_name", "package_node", "name_of_this_file"))
    ```
    In order to make this params file usable it must be executable, so lets use the following command to make it excecutable
    ```
    chmod +x Some.params
    ```



2. Add `rosparam_handler` and `dynamic_reconfigure` as dependencies in package.xml.

    ```xml
    <depend>rosparam_handler</depend>
    <depend>dynamic_reconfigure</depend>
    ```

3. Configure the CMakelists.txt for it.
  ```bash
  #########################
  ## step one
  #########################
  find_package(catkin REQUIRED COMPONENTS rosparam_handler dynamic_reconfigure)
  # include, let the generated files find the needed headers in catkin packages
  include_directories(
      include
      ${catkin_INCLUDE_DIRS}
  )
  #########################
  ## step two
  #########################
  # note that this command has to go before the catkin_package command.
  generate_ros_paramter_files(
      cfg/Some.params    ## for generating parameters header files
      #cfg/someCfgFile.cfg   ## for generating dynamic configuration files
    )
  #########################
  ## step three
  #########################
  catkin_package(
      CATKIN_DEPENDS rosparam_handler dynamic_reconfigure
  )
  # note: below commands are related to your own app.
  add_executable(example_excutable
      src/some.cpp)
  target_link_libraries(example_excutable
      ${some_libs})
  #########################
  ## step four
  #########################
  # add dependencies, note that this command should go after an example build command like above
  add_dependencies(example_executable ${PROJECT_NAME}_genparam) ## To generate SomeParamters.h file
  add_dependencies(example_executable ${PROJECT_NAME}_gencfg) ## To generate SomeConfig.h file
  ```
    Once above steps done, you should be able to generate header-only classes for parameters or cfg to use.

    Note: you can't actually find the generated header files in your package source folder. But you can see them in `catkin_ws/devel/include/package_name` for ROS or `cmake-build-debug/devel/include/package_name` for CLion like below

    ```
    SomeParameters.h  # *Parameters.h, this file will hold a struct called <name>Paramters
    SomeConfig.h      # *Config.h, this file will hold the normal dynamic_reconfigure Config struct.
    ```

4. use the generated parameter struct in your ROS node class.
    ```c++
    // include the header file
    #include "package_name/SomeParameters.h"
    // declare the instance in your class as a member
    package_name::SomeParameters param_;
    ```
    Initialize your parameters from the parameter_server (YAML file loader) via the private ROS node handle.
    ```c++
    MyNodeClass::MyNodeClass()
      : param_{ros::NodeHandle("~")} // use the private node handle. Please use getPrivateNodeHandle() for nodelets
      {
        this->param_.fromParamServer();
      }
    ```

    The `parameter_server` does not create these values of parameter from nowhere. It actually loads these values from the YAML file through the roslaunch system as follows.

    ```xml
    <?xml version="1.0"?>
    <launch>
        <arg name="config" default="$(find package_name)/launch/demo_params.yaml" />

        <node pkg="package_name" type="executable_type" name="node_name" args="" output="screen" >
            <rosparam command="load" file="$(arg config)"/>
        </node>
    </launch>
    ```
    The YMAL file may look like this.
    ```yaml
    int_param: 1
    double_param: 12.2
    bool_param: True
    vector_param: [1.0, 2.0, 3.4]
    map_param: {"a": "b", "c": "d"}
    weight: 2.3
    age: 1
    configurable_parameter: 10.
    ```

5. handy tool for modifying parameters on the fly.

    We are not done yet. If you define a parameter as a configurable object, then you can modity it on the fly just like the normal dynamic_reconfigure config parameter through rqt. You need to create a dynamic reconfiguration server to enable this feature.

    ```c++
    #include <dynamic_reconfigure/server.h>

    class MyNodeClass {
    private:
      // dynamic reconfigure,
      // the Package_name::SomeConfig is generated automatically
      dynamic_reconfigure::Server<package_name::SomeConfig> reconfigSrv_; // Dynamic reconfiguration service
      // define the callback
      void reconfigureRequest(package_name::SomeConfig&, uint32_t);
    }
    ```
    The inplementation:
    ```c++
    void MyNodeClass::reconfigureRequest(package_name::SomeConfig&, uint
      32_t level) {
        this->param_.fromConfig(config)
    }
    ```
    Set the callback function to let `dynamic_reconfigure` to modify the parameters.
    ```
    reconfigSrv_.setCallback(boost::bind(&MyNodeClass::reconfigureRequest, this, _1, _2));
    ```

      Here is what a rqt gui for dynamic_reconfigure looks like.

      ![rqt](/assets/images/rosparam_handler.png)

      Enjoy!
