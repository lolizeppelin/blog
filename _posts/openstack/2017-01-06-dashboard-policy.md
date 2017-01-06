---
layout: post
title:  "OpenStack Mitaka从零开始 dashboard使用v3版policy文件"
date:   2017-01-06 15:05:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}


dashboard不使用v3版的policy文件会有异常,role为admin的的用户不能在其他域登陆,只能登陆使用其他角色的用户（其实只要role在其他domian有使用,就不能登陆）


直接使用v3版policy文件, 又会报错无法登陆

    Jan  6 18:39:21 openstack uwsgi: Internal Server Error: /identity/
    Jan  6 18:39:21 openstack uwsgi: Traceback (most recent call last):
    Jan  6 18:39:21 openstack uwsgi: File "/usr/lib/python2.7/site-packages/django/core/handlers/base.py", line 132, in get_response
    Jan  6 18:39:21 openstack uwsgi: response = wrapped_callback(request, *callback_args, **callback_kwargs)
    Jan  6 18:39:21 openstack uwsgi: File "/usr/lib/python2.7/site-packages/horizon/decorators.py", line 36, in dec
    Jan  6 18:39:21 openstack uwsgi: return view_func(request, *args, **kwargs)
    Jan  6 18:39:21 openstack uwsgi: File "/usr/lib/python2.7/site-packages/horizon/decorators.py", line 52, in dec
    Jan  6 18:39:21 openstack uwsgi: return view_func(request, *args, **kwargs)
    Jan  6 18:39:21 openstack uwsgi: File "/usr/lib/python2.7/site-packages/horizon/decorators.py", line 36, in dec
    Jan  6 18:39:21 openstack uwsgi: return view_func(request, *args, **kwargs)
    Jan  6 18:39:21 openstack uwsgi: File "/usr/lib/python2.7/site-packages/django/views/generic/base.py", line 71, in view
    Jan  6 18:39:21 openstack uwsgi: return self.dispatch(request, *args, **kwargs)
    Jan  6 18:39:21 openstack uwsgi: File "/usr/lib/python2.7/site-packages/django/views/generic/base.py", line 89, in dispatch
    Jan  6 18:39:21 openstack uwsgi: return handler(request, *args, **kwargs)
    Jan  6 18:39:21 openstack uwsgi: File "/usr/lib/python2.7/site-packages/horizon/tables/views.py", line 159, in get
    Jan  6 18:39:21 openstack uwsgi: handled = self.construct_tables()
    Jan  6 18:39:21 openstack uwsgi: File "/usr/lib/python2.7/site-packages/horizon/tables/views.py", line 150, in construct_tables
    Jan  6 18:39:21 openstack uwsgi: handled = self.handle_table(table)
    Jan  6 18:39:21 openstack uwsgi: File "/usr/lib/python2.7/site-packages/horizon/tables/views.py", line 121, in handle_table
    Jan  6 18:39:21 openstack uwsgi: data = self._get_data_dict()
    Jan  6 18:39:21 openstack uwsgi: File "/usr/lib/python2.7/site-packages/horizon/tables/views.py", line 187, in _get_data_dict
    Jan  6 18:39:21 openstack uwsgi: self._data = {self.table_class._meta.name: self.get_data()}
    Jan  6 18:39:21 openstack uwsgi: File "./openstack_dashboard/dashboards/identity/projects/views.py", line 84, in get_data
    Jan  6 18:39:21 openstack uwsgi: self.request):
    Jan  6 18:39:21 openstack uwsgi: File "./openstack_dashboard/policy.py", line 24, in check
    Jan  6 18:39:21 openstack uwsgi: return policy_check(actions, request, target)
    Jan  6 18:39:21 openstack uwsgi: File "/usr/lib/python2.7/site-packages/openstack_auth/policy.py", line 155, in check
    Jan  6 18:39:21 openstack uwsgi: enforcer[scope], action, target, domain_credentials)
    Jan  6 18:39:21 openstack uwsgi: File "/usr/lib/python2.7/site-packages/openstack_auth/policy.py", line 169, in _check_credentials
    Jan  6 18:39:21 openstack uwsgi: if not enforcer_scope.enforce(action, target, credentials):
    Jan  6 18:39:21 openstack uwsgi: File "/usr/lib/python2.7/site-packages/oslo_policy/policy.py", line 552, in enforce
    Jan  6 18:39:21 openstack uwsgi: result = self.rules[rule](target, creds, self)
    Jan  6 18:39:21 openstack uwsgi: File "/usr/lib/python2.7/site-packages/oslo_policy/_checks.py", line 163, in __call__
    Jan  6 18:39:21 openstack uwsgi: if rule(target, cred, enforcer):
    Jan  6 18:39:21 openstack uwsgi: File "/usr/lib/python2.7/site-packages/oslo_policy/_checks.py", line 207, in __call__
    Jan  6 18:39:21 openstack uwsgi: return enforcer.rules[self.match](target, creds, enforcer)
    Jan  6 18:39:21 openstack uwsgi: File "/usr/lib/python2.7/site-packages/oslo_policy/_checks.py", line 128, in __call__
    Jan  6 18:39:21 openstack uwsgi: if not rule(target, cred, enforcer):
    Jan  6 18:39:21 openstack uwsgi: File "/usr/lib/python2.7/site-packages/oslo_policy/_checks.py", line 163, in __call__
    Jan  6 18:39:21 openstack uwsgi: if rule(target, cred, enforcer):
    Jan  6 18:39:21 openstack uwsgi: File "/usr/lib/python2.7/site-packages/oslo_policy/_checks.py", line 316, in __call__
    Jan  6 18:39:21 openstack uwsgi: return self._find_in_dict(creds, path_segments, match)
    Jan  6 18:39:21 openstack uwsgi: File "/usr/lib/python2.7/site-packages/oslo_policy/_checks.py", line 298, in _find_in_dict
    Jan  6 18:39:21 openstack uwsgi: return self._find_in_dict(test_value, path_segments, match)
    Jan  6 18:39:21 openstack uwsgi: File "/usr/lib/python2.7/site-packages/oslo_policy/_checks.py", line 289, in _find_in_dict
    Jan  6 18:39:21 openstack uwsgi: test_value = test_value[key]
    Jan  6 18:39:21 openstack uwsgi: TypeError: 'Token' object has no attribute '__getitem__'


原因在于传入的cred的token类没有
```python
    __getattr__
```

导致oslo_policy获取属性的时候报错

在这个token类在python-django-openstack-auth中,简直我操token就不能统一么Orz

我们就不去改python-django-openstack-auth了,在oslo_policy处理一下


```python
class GenericCheck(Check):
    """Check an individual match.

    Matches look like:

        - tenant:%(tenant_id)s
        - role:compute:admin
        - True:%(user.enabled)s
        - 'Member':%(role.name)s
    """

    def _find_in_dict(self, test_value, path_segments, match):
        '''Searches for a match in the dictionary.

        test_value is a reference inside the dictionary. Since the process is
        recursive, each call to _find_in_dict will be one level deeper.

        path_segments is the segments of the path to search.  The recursion
        ends when there are no more segments of path.

        When specifying a value inside a list, each element of the list is
        checked for a match. If the value is found within any of the sub lists
        the check succeeds; The check only fails if the entry is not in any of
        the sublists.

        '''

        if len(path_segments) == 0:
            return match == six.text_type(test_value)
        key, path_segments = path_segments[0], path_segments[1:]
        try:
            test_value = test_value[key]
        except KeyError:
            return False
        # patch增加特殊处理
        except TypeError:
            if hasattr(test_value, key):
              return match == six.text_type(test_value.getattr(key))
            return False
        # -----
        if isinstance(test_value, list):
            for val in test_value:
                if self._find_in_dict(val, path_segments, match):
                    return True
            return False
        else:
            return self._find_in_dict(test_value, path_segments, match)
```
