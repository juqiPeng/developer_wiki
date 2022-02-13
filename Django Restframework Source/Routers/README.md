REST框架添加了对自动URL路由到Django的支持，并为你提供了一种简单、快速和一致的方式来将视图逻辑连接到一组URL。

例子：

```python
from wiki_app.views import ListUsers
from rest_framework import routers

router = routers.SimpleRouter()
router.register(r'users', ListUsers)
urlpatterns = router.urls
```

## 1. `BaseRouter`

```python
class BaseRouter:
    def __init__(self):
        self.registry = []

    # 实例有一个 register 方法，
    def register(self, prefix, viewset, basename=None):
        """接受三个参数
        	prefix：路由的前缀
        	viewset：处理请求的viewset类
        	basename：用于创建的URL名称的基本名称。如果不设置该参数，将根据视图集的queryset属性（如果有）来自动生成基本名称。注意，如果视图集不包括queryset属性，那么在注册视图集时必须设置base_name。
        """
        if basename is None:
            basename = self.get_default_basename(viewset)
        self.registry.append((prefix, viewset, basename))

        # invalidate the urls cache
        if hasattr(self, '_urls'):
            del self._urls

    def get_default_basename(self, viewset):
        """
        If `basename` is not specified, attempt to automatically determine
        it from the viewset.
        """
        raise NotImplementedError('get_default_basename must be overridden')

    def get_urls(self):
        """
        Return a list of URL patterns, given the registered viewsets.
        """
        raise NotImplementedError('get_urls must be overridden')

    @property
    def urls(self):
        if not hasattr(self, '_urls'):
            self._urls = self.get_urls()
        return self._urls
```

BaseRouter 仿佛没有什么干货

## 2. `SimpleRouter`

```python
from collections import namedtuple

# 这里创建了两个具名元祖对象，它返回的是一个 tuple 类型的子类
# 这里将相当于创建了一个Route对象，我们通过 Route.url 访问属性
# 与元祖不同，元祖只能通过索引进行访问
Route = namedtuple('Route', ['url', 'mapping', 'name', 'detail', 'initkwargs'])
DynamicRoute = namedtuple('DynamicRoute', ['url', 'name', 'detail', 'initkwargs'])

class SimpleRouter(BaseRouter):

    # 定义多个路由器的集合
    routes = [
        # 列表路由：例如获取数据集合，创建数据集合
        Route(
            url=r'^{prefix}{trailing_slash}$',
            mapping={
                'get': 'list',
                'post': 'create'
            },
            name='{basename}-list',
            detail=False,
            initkwargs={'suffix': 'List'}
        ),
        # 动态列表路由
        # 例如：viewset中的装饰器：@action(detail=False)
        DynamicRoute(
            url=r'^{prefix}/{url_path}{trailing_slash}$',
            name='{basename}-{url_name}',
            detail=False,
            initkwargs={}
        ),
        # 单数据路由：例如获取单条数据，修改单条数据，删除单条数据
        Route(
            url=r'^{prefix}/{lookup}{trailing_slash}$',
            mapping={
                'get': 'retrieve',
                'put': 'update',
                'patch': 'partial_update',
                'delete': 'destroy'
            },
            name='{basename}-detail',
            detail=True,
            initkwargs={'suffix': 'Instance'}
        ),
        # 动态数据路由
        # # 例如：viewset中的装饰器：@action(detail=True)
        DynamicRoute(
            url=r'^{prefix}/{lookup}/{url_path}{trailing_slash}$',
            name='{basename}-{url_name}',
            detail=True,
            initkwargs={}
        ),
    ]

    def __init__(self, trailing_slash=True):
        self.trailing_slash = '/' if trailing_slash else ''
        super().__init__()

    def get_default_basename(self, viewset):
        """
        自动生成 basename
        """
        queryset = getattr(viewset, 'queryset', None)

        assert queryset is not None, '`basename` argument not specified, and could ' \
            'not automatically determine the name from the viewset, as ' \
            'it does not have a `.queryset` attribute.'
				# 默认使用 queryset 绑定的模型类的小写名
        return queryset.model._meta.object_name.lower()

    def get_routes(self, viewset):
        """
        Augment `self.routes` with any dynamically generated routes.

        Returns a list of the Route namedtuple.
        """
        # 获取预设的所有action：
        # 在上面我们可以看到默认预设的action有：['list', 'create', 'retrieve', 'update', 'partial_update', 'destroy']
        known_actions = list(flatten([route.mapping.values() for route in self.routes if isinstance(route, Route)]))
        
        # 这个是我们在 viewset 中通过 @action 装饰器添加的 action
        extra_actions = viewset.get_extra_actions()

        # action 那么是否出现在 known_actions 预设的 action 中
        # 如果已经出现，那么将会抛出异常
        not_allowed = [
            action.__name__ for action in extra_actions
            if action.__name__ in known_actions
        ]
        if not_allowed:
            msg = ('Cannot use the @action decorator on the following '
                   'methods, as they are existing routes: %s')
            raise ImproperlyConfigured(msg % ', '.join(not_allowed))

        # 根据 @action 装饰器中 detail 的属性分成两类
        detail_actions = [action for action in extra_actions if action.detail]
        list_actions = [action for action in extra_actions if not action.detail]

        # 重新添加所有的 action 
        routes = []
        for route in self.routes:
            if isinstance(route, DynamicRoute) and route.detail:
                routes += [self._get_dynamic_route(route, action) for action in detail_actions]
            elif isinstance(route, DynamicRoute) and not route.detail:
                routes += [self._get_dynamic_route(route, action) for action in list_actions]
            else:
                routes.append(route)

        return routes

    def _get_dynamic_route(self, route, action):
        initkwargs = route.initkwargs.copy()
        initkwargs.update(action.kwargs)

        url_path = escape_curly_brackets(action.url_path)

        return Route(
            url=route.url.replace('{url_path}', url_path),
            mapping=action.mapping,
            name=route.name.replace('{url_name}', action.url_name),
            detail=route.detail,
            initkwargs=initkwargs,
        )

    def get_method_map(self, viewset, method_map):
        """
					只返回 viewset 中实现的 action
        """
        bound_methods = {}
        for method, action in method_map.items():
            if hasattr(viewset, action):
                bound_methods[method] = action
        return bound_methods

    def get_lookup_regex(self, viewset, lookup_prefix=''):
        """
        给定一个视图集，返回用于匹配单个实例的 URL 正则表达式部分
       	这一部分主要是用于类似：/users/100/ 这样，匹配 id 为 100 的 user 实例
        """
        base_regex = '(?P<{lookup_prefix}{lookup_url_kwarg}>{lookup_value})'
				
        # 默认是使用模型类的 pk 字段进行匹配
        lookup_field = getattr(viewset, 'lookup_field', 'pk')
        lookup_url_kwarg = getattr(viewset, 'lookup_url_kwarg', None) or lookup_field
        lookup_value = getattr(viewset, 'lookup_value_regex', '[^/.]+')
        # 返回一个正则表达式
        # 例如：'(?P<pk>[^/.]+)'
        return base_regex.format(
            lookup_prefix=lookup_prefix,
            lookup_url_kwarg=lookup_url_kwarg,
            lookup_value=lookup_value
        )

    def get_urls(self):
        """
        当我们调用register方法的时候，会将我们的路由注册到 self.registry 属性中
        """
        ret = []
				
        for prefix, viewset, basename in self.registry:
            lookup = self.get_lookup_regex(viewset)  # 匹配单个实例的正则表达式
            routes = self.get_routes(viewset)  # 装载所有的 action 

            for route in routes:

                # 只有视图集中真正存在的action才会被绑定
                mapping = self.get_method_map(viewset, route.mapping)
                if not mapping:
                    continue

                # Build the url pattern
                regex = route.url.format(
                    prefix=prefix,
                    lookup=lookup,
                    trailing_slash=self.trailing_slash
                )

                # If there is no prefix, the first part of the url is probably
                #   controlled by project's urls.py and the router is in an app,
                #   so a slash in the beginning will (A) cause Django to give
                #   warnings and (B) generate URLS that will require using '//'.
                # 做一些前缀的处理
                if not prefix and regex[:2] == '^/':
                    regex = '^' + regex[2:]

                initkwargs = route.initkwargs.copy()
                initkwargs.update({
                    'basename': basename,
                    'detail': route.detail,
                })

                # 调用视图集的 as_view 方法， 这里就是我们熟悉的环节了，不用多做赘述
                view = viewset.as_view(mapping, **initkwargs)
                name = route.name.format(basename=basename)
                ret.append(re_path(regex, view, name=name))

        return ret
```

## 3. 使用

```python
from rest_framework import routers

router = routers.SimpleRouter()
router.register(r'users', UserViewSet)
router.register(r'accounts', AccountViewSet)
urlpatterns = router.urls
```

```python
# 上面的将会生成这样一个路由模式列表

"""
URL pattern: ^users/$ Name: 'user-list'
URL pattern: ^users/{pk}/$ Name: 'user-detail'
URL pattern: ^accounts/$ Name: 'account-list'
URL pattern: ^accounts/{pk}/$ Name: 'account-detail'
"""
```

### 3.1 添加额外路由`detail_route`

```python
from myapp.permissions import IsAdminOrIsSelf
from rest_framework.decorators import action

class UserViewSet(ModelViewSet):
    ...
		
    # 如果没有指定 url_path 参数，那么将会自己生成前缀：set_password（默认采用方法名）
    @action(methods=['post'], detail=False, permission_classes=[IsAdminOrIsSelf], url_path='change-password')
    def set_password(self, request, pk=None):
        ...
```

```python
# 上面的例子将会生成这样一个路由模式列表
"""
URL pattern: ^users/$ Name: 'user-list'
URL pattern: ^users/{pk}/$ Name: 'user-detail'
URL pattern: ^users/{pk}/change-password/$ Name: 'user-change-password'  (当detail=True时)
URL pattern: ^users/change-password/$ Name: 'user-change-password'  (当detail=False时)
URL pattern: ^accounts/$ Name: 'account-list'
URL pattern: ^accounts/{pk}/$ Name: 'account-detail'
"""
```

### 3.2 关于尾部斜杠

默认情况下，由`SimpleRouter`创建的URL将附加尾部斜杠。 在实例化路由器时，可以通过将`trailing_slash`参数设置为`False'来修改此行为。比如：

```python
router = SimpleRouter(trailing_slash=False)
```

这里有个小坑，当我们没有指定这个参数的时候，假设我们的请求的url没有加上 / 结尾，对于 get 请求，将会自动加上，但是对于 post 则不会，并且会抛出异常。所以这里是个需要注意的地方