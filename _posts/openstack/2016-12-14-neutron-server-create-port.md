---
layout: post
title:  "OpenStack Mitaka从零开始 nova通过neutron分配网络的过程(2)"
date:   2016-12-14 15:05:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}

```python
class APIParamsCall(object):
    """A Decorator to support formatting and tenant overriding and filters."""
    def __init__(self, function):
        self.function = function

    def __get__(self, instance, owner):
        def with_params(*args, **kwargs):
            _format = instance.format
            if 'format' in kwargs:
                instance.format = kwargs['format']
            ret = self.function(instance, *args, **kwargs)
            instance.format = _format
            return ret
        return with_params
```


```python

def get(self, action, body=None, headers=None, params=None):
    return self.retry_request("GET", action, body=body,
                              headers=headers, params=params)

def list(self, collection, path, retrieve_all=True, **params):
    if retrieve_all:
        res = []
        request_ids = []
        for r in self._pagination(collection, path, **params):
            res.extend(r[collection])
            request_ids.extend(r.request_ids)
        return _DictWithMeta({collection: res}, request_ids)
    else:
        return _GeneratorWithMeta(self._pagination, collection,
                                  path, **params)

def _pagination(self, collection, path, **params):
    if params.get('page_reverse', False):
        linkrel = 'previous'
    else:
        linkrel = 'next'
    next = True
    while next:
        res = self.get(path, params=params)
        yield res
        next = False
        try:
            for link in res['%s_links' % collection]:
                if link['rel'] == linkrel:
                    query_str = urlparse.urlparse(link['href']).query
                    params = urlparse.parse_qs(query_str)
                    next = True
                    break
        except KeyError:
            break

@APIParamsCall
def list_ports(self, retrieve_all=True, **_params):
    """Fetches a list of all ports for a tenant."""
    # Pass filters in "params" argument to do_request
    return self.list('ports', self.ports_path, retrieve_all,
                     **_params)

```


neutron server分配创建port

    文件neutron.api.v2.base.py

```python
# collection ports
# resource port

def create(self, request, body=None, **kwargs):
    self._notifier.info(request.context,
                        self._resource + '.create.start',
                        body)
    return self._create(request, body, **kwargs)


def _create(self, request, body, **kwargs):
    """Creates a new instance of the requested entity."""
    parent_id = kwargs.get(self._parent_id_name)
    body = Controller.prepare_request_body(request.context,
                                           copy.deepcopy(body), True,
                                           self._resource, self._attr_info,
                                           allow_bulk=self._allow_bulk)
    action = self._plugin_handlers[self.CREATE]
    # Check authz
    if self._collection in body:
        # Have to account for bulk create
        items = body[self._collection]
    else:
        items = [body]
    # Ensure policy engine is initialized
    policy.init()
    # Store requested resource amounts grouping them by tenant
    # This won't work with multiple resources. However because of the
    # current structure of this controller there will hardly be more than
    # one resource for which reservations are being made
    request_deltas = collections.defaultdict(int)
    for item in items:
        self._validate_network_tenant_ownership(request,
                                                item[self._resource])
        policy.enforce(request.context,
                       action,
                       item[self._resource],
                       pluralized=self._collection)
        if 'tenant_id' not in item[self._resource]:
            # no tenant_id - no quota check
            continue
        tenant_id = item[self._resource]['tenant_id']
        request_deltas[tenant_id] += 1
    # Quota enforcement
    reservations = []
    try:
        for (tenant, delta) in request_deltas.items():
            reservation = quota.QUOTAS.make_reservation(
                request.context,
                tenant,
                {self._resource: delta},
                self._plugin)
            reservations.append(reservation)
    except exceptions.QuotaResourceUnknown as e:
            # We don't want to quota this resource
            LOG.debug(e)

    def notify(create_result):
        # Ensure usage trackers for all resources affected by this API
        # operation are marked as dirty
        with request.context.session.begin():
            # Commit the reservation(s)
            for reservation in reservations:
                quota.QUOTAS.commit_reservation(
                    request.context, reservation.reservation_id)
            resource_registry.set_resources_dirty(request.context)

        notifier_method = self._resource + '.create.end'
        self._notifier.info(request.context,
                            notifier_method,
                            create_result)
        self._send_dhcp_notification(request.context,
                                     create_result,
                                     notifier_method)
        return create_result

    def do_create(body, bulk=False, emulated=False):
        kwargs = {self._parent_id_name: parent_id} if parent_id else {}
        if bulk and not emulated:
            obj_creator = getattr(self._plugin, "%s_bulk" % action)
        else:
            obj_creator = getattr(self._plugin, action)
        try:
            if emulated:
                return self._emulate_bulk_create(obj_creator, request,
                                                 body, parent_id)
            else:
                if self._collection in body:
                    # This is weird but fixing it requires changes to the
                    # plugin interface
                    kwargs.update({self._collection: body})
                else:
                    kwargs.update({self._resource: body})
                return obj_creator(request.context, **kwargs)
        except Exception:
            # In case of failure the plugin will always raise an
            # exception. Cancel the reservation
            with excutils.save_and_reraise_exception():
                for reservation in reservations:
                    quota.QUOTAS.cancel_reservation(
                        request.context, reservation.reservation_id)

    if self._collection in body and self._native_bulk:
        # plugin does atomic bulk create operations
        objs = do_create(body, bulk=True)
        # Use first element of list to discriminate attributes which
        # should be removed because of authZ policies
        fields_to_strip = self._exclude_attributes_by_policy(
            request.context, objs[0])
        return notify({self._collection: [self._filter_attributes(
            request.context, obj, fields_to_strip=fields_to_strip)
            for obj in objs]})
    else:
        if self._collection in body:
            # Emulate atomic bulk behavior
            objs = do_create(body, bulk=True, emulated=True)
            return notify({self._collection: objs})
        else:
            obj = do_create(body)
            self._send_nova_notification(action, {},
                                         {self._resource: obj})
            return notify({self._resource: self._view(request.context,
                                                      obj)})

```
