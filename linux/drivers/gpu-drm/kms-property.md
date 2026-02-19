DRM property
============

## Property

- a `drm_property` is a `drm_mode_object` of `DRM_MODE_OBJECT_PROPERTY`
  - it has type, flags, and allowed values
  - it does not have the current value!
- `drm_object_attach_property` instantiates a property
  - the instance has the current value!
- property types
  - `DRM_MODE_PROP_RANGE` with `drm_property_create_range`
  - `DRM_MODE_PROP_SIGNED_RANGE` with `drm_property_create_signed_range`
  - `DRM_MODE_PROP_ENUM` with `drm_property_create_enum`
  - `DRM_MODE_PROP_BITMASK` with `drm_property_create_bitmask`
  - `DRM_MODE_PROP_OBJECT` with `drm_property_create_object`
  - `DRM_MODE_PROP_BLOB` with `drm_property_create`
- property flags
  - `DRM_MODE_PROP_ATOMIC`
  - `DRM_MODE_PROP_IMMUTABLE`
- most properties are global
  - they are in the per-device `drm_mode_config`
- some properties are per-plane, per-crtc, or per-connector
  - this is because each property has allowed values
  - only when two otherwise identical properties have different allowed
    values, such as when two planes have two different caps, they are made
    per-object

## KMS object properties

- `DRM_IOCTL_MODE_OBJ_GETPROPERTIES` can get all properties of a KMS object
  - in: object id and object type `DRM_MODE_OBJECT_*`
  - out: an array of (32-bit prop id, 64-bit prop val) pairs
  - each prop id seems to be globally unique (see below)
- `DRM_IOCTL_MODE_GETPROPERTY` can get info about a prop id
  - in: prop id
  - out: name and values
    - name is well-defined and is a part of the API
    - values depends on the flags
      - for `DRM_MODE_PROP_RANGE`, values are 64-bit pairs
      - for `DRM_MODE_PROP_ENUM`, values are an array of
        (well-defined enum name, 64-bit enum id)
      - for `DRM_MODE_PROP_BITMASK` is similar to `DRM_MODE_PROP_ENUM`
      - for `DRMO_MODE_PROP_BLOB`, values are an array of 32-bit blob ids that
      	can be passed to `DRM_IOCTL_MODE_GETPROPBLOB`
  - remember that these are property infos; the current value is returned by
    `DRM_IOCTL_MODE_OBJ_GETPROPERTIES`
- for example, a `DRM_MODE_OBJECT_PLANE` object has a property `type`
  indicating whether the plane is primary, cursor, or overlay
  - we list all properties first to get prop ids and prop values first
  - for each prop id, we get the info.  We are interested in the prop id whose
    name is `type`
  - the info should also contains a list of (enum name, enum val) enums
  - by comparing the prop value with the enum vals, we can find out the
    current enum name, which should be one of `Primary`, `Overlay`, or
    `Cursor`
