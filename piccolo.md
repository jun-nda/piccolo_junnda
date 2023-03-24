1.反射序列化代码生成

2.使用上一步生成的代码序列化/反序列化资产，形成资产对象

由于上一步我们看到在createObject的时候走到了加载资产这里，所以我们从这里看一下资产是如何被加载到内存并整理成C++对象的。

```C++
        template<typename AssetType>
        bool loadAsset(const std::string& asset_url, AssetType& out_asset) const
        {
            // read json file to string
            std::filesystem::path asset_path = getFullPath(asset_url);
            std::ifstream asset_json_file(asset_path);
            if (!asset_json_file)
            {
                LOG_ERROR("open file: {} failed!", asset_path.generic_string());
                return false;
            }

            std::stringstream buffer;
            buffer << asset_json_file.rdbuf();
            std::string asset_json_text(buffer.str());

            // parse to json object and read to runtime res object
            std::string error;
            auto&&      asset_json = Json::parse(asset_json_text, error);
            if (!error.empty())
            {
                LOG_ERROR("parse json file {} failed!", asset_url);
                return false;
            }

            Serializer::read(asset_json, out_asset);
            return true;
        }
```

这里首先语法上用到了模板，最简单的那种，所以不多赘述。

逻辑上我们先获取绝对路径，然后使用std的文件输入流，把文件转化成字符串，再用Json库传换成Json的字符串。

一切都准备好了之后，开始反序列化!

反序列化这里我们就看反序列化的逻辑，先不要纠结反序列化的代码是怎么生成的，这个比较复杂，但是没关系后面我们会继续更新。

这里反序列化我们最好对着资产文件来看。

```C++
        if(!json_context["components"].is_null()){
            assert(json_context["components"].is_array());
            Json::array array_m_components = json_context["components"].array_items();
            instance.m_components.resize(array_m_components.size());
            for (size_t index=0; index < array_m_components.size();++index){
                Serializer::read(array_m_components[index], instance.m_components[index]);
            }
        }
```

```json
{
  "components": [
    {
      "$context": {
        "particle_res": {
          "local_translation": {
            "x": 0,
            "y": 0,
            "z": 0
          },
          "velocity": {
            "x": 0.02,
            "y": 0.02,
            "z": 2.5,
            "w": 4.0
          },
          "acceleration": {
            "x": 0.0,
            "y": 0.0,
            "z": -2.5,
            "w": 0.0
          },
          "size": {
            "x": 0.02,
            "y": 0.02,
            "z": 0.00
          },
          "emitter_type": 1,
          "life": {
            "x": 1.5,
            "y": 0.0
          },
          "color": {
            "x": 9.047718,
            "y": 0.601811,
            "z": 0.0,
            "w": 1.0
          }
        }
      },
      "$typeName": "ParticleComponent"
    }
  ]
}
```

逻辑很简单吧，就是遍历然后再继续反序列化每一个`component`

注意这里vector的resize是会直接初始化对象的，这是个常用的小技巧哦。

然后继续看。

```C++
        template<typename T>
        static T*& read(const Json& json_context, Reflection::ReflectionPtr<T>& instance)
        {
            std::string type_name = json_context["$typeName"].string_value();
            instance.setTypeName(type_name);
            return readPointer(json_context, instance.getPtrReference());
        }
```

通过资产文件我们可以看到该资产只有一个 `component`，所以这里就是反序列化的代码，我们看到这里先记了一下 `typename`，然后我们继续看看 `readPointer`里做了什么。

```C++
        static T*& readPointer(const Json& json_context, T*& instance)
        {
            assert(instance == nullptr);
            std::string type_name = json_context["$typeName"].string_value();
            assert(!type_name.empty());
            if ('*' == type_name[0])
            {
                instance = new T;
                read(json_context["$context"], *instance);
            }
            else
            {
                instance = static_cast<T*>(
                    Reflection::TypeMeta::newFromNameAndJson(type_name, json_context["$context"]).m_instance);
            }
            return instance;
        }
```

这里很有意思，感觉发现了新大陆，这里判断了一下type_name的第一个字符是不是'*'，难道是判断是不是指针?what? 我们怀着激动的心情继续往下看。

```C++
        ReflectionInstance TypeMeta::newFromNameAndJson(std::string type_name, const Json& json_context)
        {
            auto iter = m_class_map.find(type_name);

            if (iter != m_class_map.end())
            {
                return ReflectionInstance(TypeMeta(type_name), (std::get<1>(*iter->second)(json_context)));
            }
            return ReflectionInstance();
        }
```

m_class_map 这是哪来的? 为啥它保存了所有类的名字和一个叫什么Tuple的东西，这是啥? 又要学新~~姿势~~知识了，好激动。

想知道它是干嘛的，就看看它在哪里被赋值的咯，我们找了一圈最后找到了这里。

```C++
        void TypeMetaRegisterinterface::registerToClassMap(const char* name, ClassFunctionTuple* value)
        {
            if (m_class_map.find(name) == m_class_map.end())
            {
                m_class_map.insert(std::make_pair(name, value));
            }
            else
            {
                delete value;
            }
        }
```

逻辑看起来应该是，根据名字对应key，如果没有key，则把name和value组成pair插进去，如果没有，则把传进来的value资源释放。

只看这里看不出啥，看看哪里调的他

![1679645955529](image/piccolo/1679645955529.png)

原来是调它的地方太多直接写成宏了，那就继续看看吧，结果发现这个宏遍布了所有被自动生成的反射文件中。

在这里纠结一下是继续深究还是回去看使用的地方。

这里做的事情应该叫做注册元数据，是在引擎运行一开始就注册了的，噢那没事了，知道是一开始注册了就行了，那么我们回到刚才用的地方继续看。

`return ReflectionInstance(TypeMeta(type_name), (std::get<1>(*iter->second)(json_context)));`

这句话，很关键，仔细看看啥意思。

首先是返回一个 `ReflectionInstance`对象，`ReflectionInstance`的构造函数长这样：

`ReflectionInstance(TypeMeta meta, void* instance) : m_meta(meta), m_instance(instance) {}`

好的，第一个参数显而易见，那第二个参数啥情况？

首先 std::get<>的意思是拿容器里的第几个元素，所以意思是，返回 `(*iter->second)`的第一个元素。

然后呢，后面还有个括号，看起来前面拿到的应该是一个函数句柄，看看 `(*iter->second)`到底是是个啥

```C++
static std::map<std::string, ClassFunctionTuple*>       m_class_map;
```

那 `ClassFunctionTuple`又是个啥？

```c++
typedef std::tuple<GetBaseClassReflectionInstanceListFunc, ConstructorWithJson, WriteJsonByName> ClassFunctionTuple;
```

嗯...我知道可能有的小可爱现在已经晕了，但是先别晕，我们先要明确 `std::tuple`是干啥的，为啥这里要用它。

好吧，C++Primer表示，tuple可以简单理解为：“快速而随意的数据结构”，并且开头就说tuple是类似pair的模板，

那我觉得完全就可以按照pair的思路来理解了，并且长得和pair还挺像的哈哈。

那我们看看tuple里这仨都是啥。

```C++
typedef std::function<void*(const Json&)>                           ConstructorWithJson;
typedef std::function<Json(void*)>                                  WriteJsonByName;
typedef std::function<int(Reflection::ReflectionInstance*&, void*)> GetBaseClassReflectionInstanceListFunc;
```

嗯，果然是三个function，好的，那我们此时回来，

`(std::get<1>(*iter->second)(json_context))`，这回再看看这是啥，应该能知道了，

实际上是，std::function<void*(const Json&)>(json_context)

好，那么现在的问题是什么，当然是上面整个std::function究竟做了些什么，我们需要找到给这个function赋值的地方。

好的，还是之前 `registerToClassMap`的地方,然后也是被宏包着的，那我们试着找到ParticleComponent定义得地方。

```C++
        ClassFunctionTuple* class_function_tuple_ParticleComponent=new ClassFunctionTuple(
            &TypeFieldReflectionOparator::TypeParticleComponentOperator::getParticleComponentBaseClassReflectionInstanceList,
            &TypeFieldReflectionOparator::TypeParticleComponentOperator::constructorWithJson,
            &TypeFieldReflectionOparator::TypeParticleComponentOperator::writeByName);
        REGISTER_BASE_CLASS_TO_MAP("ParticleComponent", class_function_tuple_ParticleComponent);
```

终于找到了，就是它，我们需要看的就是第二个函数。

```c++
        static void* constructorWithJson(const Json& json_context){
            ParticleComponent* ret_instance= new ParticleComponent;
            Serializer::read(json_context, *ret_instance);
            return ret_instance;
        }
```

好，我们创建了一个ParticleComponent，然后呢，然后又进行了反序列化???

好吧，这里应该是递归地去反序列化组件了。

有点晕，纠结一下要不要继续往下看......纠结完了，继续往下刨根问底，干就完了！

```c++
    ParticleComponent& Serializer::read(const Json& json_context, ParticleComponent& instance){
        assert(json_context.is_object());
        Serializer::read(json_context,*(Piccolo::Component*)&instance);
        if(!json_context["particle_res"].is_null()){
            Serializer::read(json_context["particle_res"], instance.m_particle_res);
        }
        return instance;
    }
```

这里第一个read进去发现啥也没干，猜测是给什么功能扩充用的，mini版删掉了。

重点是第二个read，它将会继续对下一层进行反序列化。

```c++
    template<>
    ParticleComponentRes& Serializer::read(const Json& json_context, ParticleComponentRes& instance){
        assert(json_context.is_object());
      
        if(!json_context["local_translation"].is_null()){
            Serializer::read(json_context["local_translation"], instance.m_local_translation);
        }
        if(!json_context["local_rotation"].is_null()){
            Serializer::read(json_context["local_rotation"], instance.m_local_rotation);
        }
        if(!json_context["velocity"].is_null()){
            Serializer::read(json_context["velocity"], instance.m_velocity);
        }
        if(!json_context["acceleration"].is_null()){
            Serializer::read(json_context["acceleration"], instance.m_acceleration);
        }
        if(!json_context["size"].is_null()){
            Serializer::read(json_context["size"], instance.m_size);
        }
        if(!json_context["emitter_type"].is_null()){
            Serializer::read(json_context["emitter_type"], instance.m_emitter_type);
        }
        if(!json_context["life"].is_null()){
            Serializer::read(json_context["life"], instance.m_life);
        }
        if(!json_context["color"].is_null()){
            Serializer::read(json_context["color"], instance.m_color);
        }
        return instance;
    }
```

好的，我们终于进入到了反序列化树状结构的倒数第二层节点，对照着json文件，这里已经很清晰了，就是一一对应赋值。

在下面就是叶子节点了，这....噢还不是叶子节点

```c++
    Vector3& Serializer::read(const Json& json_context, Vector3& instance){
        assert(json_context.is_object());
      
        if(!json_context["x"].is_null()){
            Serializer::read(json_context["x"], instance.x);
        }
        if(!json_context["y"].is_null()){
            Serializer::read(json_context["y"], instance.y);
        }
        if(!json_context["z"].is_null()){
            Serializer::read(json_context["z"], instance.z);
        }
        return instance;
    }
```

这个才是叶子节点，好吧问题不大。

```c++
    float& Serializer::read(const Json& json_context, float& instance)
    {
        assert(json_context.is_number());
        return instance = static_cast<float>(json_context.number_value());
    }
```

好的，那么到这里为止，我们已经梳理完了反序列化树状结构的其中一条分支。

这是一条非常普通的分支，所以具有共性，其他的大多数分支也是大同小异；但是很失望的一点是我们没有看到指针的序列化，现在暂时还不知道是没有针对指针进行实现还是怎么，后面看一下，如果有指针的序列化那将是绝杀。（2023/03/24）


3.使用资产对象生成场景对象

全局只有一个地方可以创建对象，在 `Level`里，Level我们在Hazel引擎中学过，是用来场景的分层管理的一个抽象层。所以外层应该是遍历每一个Level然后根据资源创建每一个 `Object。`

值得注意的是，这里Object的Id是每次运行动态分配的，并没有序列化到文件里，这就意味着假如一个Object在编辑器或者运行时有其他Object引用他，那关闭程序再次运行时，那个Obect将会失去对该Object的引用，鉴于piccolo只是一个mini引擎，所以这里暂时不深入讨论。

```c++
    GObjectID Level::createObject(const ObjectInstanceRes& object_instance_res)
    {
        GObjectID object_id = ObjectIDAllocator::alloc();
        ASSERT(object_id != k_invalid_gobject_id);

        std::shared_ptr<GObject> gobject;
        try
        {
            gobject = std::make_shared<GObject>(object_id);
        }
        catch (const std::bad_alloc&)
        {
            LOG_FATAL("cannot allocate memory for new gobject");
        }

        bool is_loaded = gobject->load(object_instance_res);
        if (is_loaded)
        {
            m_gobjects.emplace(object_id, gobject);
        }
        else
        {
            LOG_ERROR("loading object " + object_instance_res.m_name + " failed");
            return k_invalid_gobject_id;
        }
        return object_id;
    }
```

可以看到这里最重要的一行代码是这句 `bool is_loaded = gobject->load(object_instance_res);`，将传进来的资源对象的数据传给我们的 `GObject`，也就是场景里渲染真正要用的对象。

然后我们进到load函数里面去看看怎么事儿。

```c++
    bool GObject::load(const ObjectInstanceRes& object_instance_res)
    {
        // clear old components
        m_components.clear();

        setName(object_instance_res.m_name);

        // load object instanced components
        m_components = object_instance_res.m_instanced_components;
        for (auto component : m_components)
        {
            if (component)
            {
                component->postLoadResource(weak_from_this());
            }
        }

        // load object definition components
        m_definition_url = object_instance_res.m_definition;

        ObjectDefinitionRes definition_res;

        const bool is_loaded_success = g_runtime_global_context.m_asset_manager->loadAsset(m_definition_url, definition_res);
        if (!is_loaded_success)
            return false;

        for (auto loaded_component : definition_res.m_components)
        {
            const std::string type_name = loaded_component.getTypeName();
            // don't create component if it has been instanced
            if (hasComponent(type_name))
                continue;

            loaded_component->postLoadResource(weak_from_this());

            m_components.push_back(loaded_component);
        }

        return true;
    }
```

可以看到基本上是一个一对一赋值的过程，我们首先要看看Obect里有啥东西。

```C++
    protected:
        GObjectID   m_id {k_invalid_gobject_id};
        std::string m_name;
        std::string m_definition_url;

        // we have to use the ReflectionPtr due to that the components need to be reflected 
        // in editor, and it's polymorphism
        std::vector<Reflection::ReflectionPtr<Component>> m_components;
```

噢，原来是一个全局自增的ID，一个名字，一个定义url（这个暂时不知道干啥的，后面看），然后看起来是一个比较重要的用反射指针包着的组件群（在piccolo中要使用反射功能的组件都要这样包起来，具体原因参考第一节的反射序列化详解）。

所以我们回到 `Object::load`来，这里先格式化一下组件容器，然后设置名字，接下来是 `load components`，这里为什么还要加载组件呢，我们进去看看，直接F12是看不到的，因为是个虚函数，在运行时才能确认，所以我们运行一下看看。

断点看到一个 `TransformComponent`组件，这里我们需要传一个Object的 weak_ptr进来，emmm先暂停一下，为什么这里要使用weak_ptr呢？或许是一个小知识点，我们来看看。

语法当然是不必讲了，我们主要要清楚，为啥要有weak_ptr这东西。

经过查资料我们发现，首先很常用的一点就是，避免Shared_Ptr相互引用导致内存泄露。

嗯，具体用法整理到这里了。[C++11 -- 智能指针 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/616643027)

好的我们继续回来整理这块的逻辑，可以看到这里将是为了给component记一下自己的父object是谁，然后记了一个变换的buffer，不知道是干啥的，后面再看，然后还有一个dirty，这个是Component基类里的属性，大胆猜测是用来记录是否初始化过的，后面再看。

```C++
    void TransformComponent::postLoadResource(std::weak_ptr<GObject> parent_gobject)
    {
        m_parent_object       = parent_gobject;
        m_transform_buffer[0] = m_transform;
        m_transform_buffer[1] = m_transform;
        m_is_dirty            = true;
    }
```

总之这里就是给commpoent记了一些数据，然后我们继续往下看。

`m_definition_url = object_instance_res.m_definition;`，哟，记了一下资源路径，这是啥意思捏。

噢，原来紧接着下面就开始走loadAsset了，可能这个路径后面有用处呗。

```C++
ObjectDefinitionRes definition_res;

        const bool is_loaded_success = g_runtime_global_context.m_asset_manager->loadAsset(m_definition_url, definition_res);
```

(2023/03/24)

好的，我们梳理完了一条反序列化的流程之后，回到这里，definition_res就是反序列化进行完毕我们拿到的数据(资产对象),

继续往下看。

```c++
        for (auto loaded_component : definition_res.m_components)
        {
            const std::string type_name = loaded_component.getTypeName();
            // don't create component if it has been instanced
            if (hasComponent(type_name))
                continue;

            loaded_component->postLoadResource(weak_from_this());

            m_components.push_back(loaded_component);
        }
```

等等，这里咋又调了一次 `postLoadResource`，啥情况？ 再仔细看看呢？

噢，原来是对刚才加载好的资产对象的component进行处理，处理好之后再添加到m_components里。

基类里给的`postLoadResource`的注释是`// Instantiating the component after definition loaded`

实例化组件? 为啥还要实例化，不是构造函数执行完就实例化了么？

注意后半句：在 definition load之后，意思就是说，一开始我们先创建对象，但是因为没有加载资源，我们的对象是空的，等加载完资源之后，我们再进行真正的实例化，妙啊。

m_particle_res是哪来的？ 噢，应该是反序列化的时候，中间不知道那一层给赋上的值，今天先到这里，下回再继续（已经有点发晕了，但是要加油!）


4.场景对象的渲染和交互
