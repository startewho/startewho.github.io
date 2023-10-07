windowns-rs是rust-com的进化版。
提供了原生的windows api，包括winrt.
取消了自定义com组件的功能

## Interface
`Interface`使用了rust的trait.

```rust
pub unsafe trait Interface: Sized {
    type Vtable;

    /// A reference to the interface's vtable
    #[doc(hidden)]
    fn vtable(&self) -> &Self::Vtable {
        // SAFETY: the implementor of the trait guarantees that `Self` is castable to its vtable
        unsafe { self.assume_vtable::<Self>() }
    }

}

```


我们com接口都需要实现Interface的trait。
## ComInterface

```rust
pub unsafe trait ComInterface: Interface + Clone {
    /// A unique identifier representing this interface.
    const IID: GUID;

    // Casts the `ComInterface` to a `IUnknown`.
    fn as_unknown(&self) -> &IUnknown {
        // SAFETY: it is always safe to treat a `ComInterface` as an `IUnknown`.
        unsafe { std::mem::transmute(self) }
    }

fn cast<T: ComInterface>(&self) -> Result<T> {
        let mut result = None;
        // SAFETY: `result` is valid for writing an interface pointer and it is safe
        // to cast the `result` pointer as `T` on success because we are using the `IID` tied
        // to `T` which the implementor of `ComInterface` has guaranteed is correct
        unsafe { self.query(&T::IID, &mut result as *mut _ as _).and_some(result) }
    }

 /// Call `QueryInterface` on this interface
    ///
    /// # Safety
    ///
    /// `interface` must be a non-null, valid pointer for writing an interface pointer.
    unsafe fn query(&self, iid: *const GUID, interface: *mut *mut std::ffi::c_void) -> HRESULT {
        (self.assume_vtable::<IUnknown>().QueryInterface)(self.as_raw(), iid, interface)
    }
```

ComInterface继承与Intreface。多了Com的特性IID,还有最关键的cast方法可以调用Com的QueryInteface来获取对应接口


## IUnkonwn

```rust
/// All COM interfaces (and thus WinRT classes and interfaces) implement
/// [IUnknown](https://docs.microsoft.com/en-us/windows/win32/api/unknwn/nn-unknwn-iunknown)
/// under the hood to provide reference-counted lifetime management as well as the ability
/// to query for additional interfaces that the object may implement.
#[repr(transparent)]
pub struct IUnknown(std::ptr::NonNull<std::ffi::c_void>);

#[doc(hidden)]
#[repr(C)]
pub struct IUnknown_Vtbl {
    pub QueryInterface: unsafe extern "system" fn(this: *mut std::ffi::c_void, iid: *const GUID, interface: *mut *mut std::ffi::c_void) -> HRESULT,
    pub AddRef: unsafe extern "system" fn(this: *mut std::ffi::c_void) -> u32,
    pub Release: unsafe extern "system" fn(this: *mut std::ffi::c_void) -> u32,
}

```

我们所有的com api都需要包含（或者间接保护此struct）。

IUnkonwn包含一个指针，指针指向我们调用的方法表

## 例子

我们用Uri组件说明：

```
#[doc = "*Required features: `\"Win32_System_Com\"`*"]
#[repr(transparent)]
#[derive(::core::cmp::PartialEq, ::core::cmp::Eq, ::core::fmt::Debug, ::core::clone::Clone)]
pub struct IUri(::windows_core::IUnknown);
```
我们IUri。包含了IUnknown,接着实现方法表
```rust
#[repr(C)]
#[doc(hidden)]
pub struct IUri_Vtbl {
    pub base__: ::windows_core::IUnknown_Vtbl,
    pub GetPropertyBSTR: unsafe extern "system" fn(this: *mut ::core::ffi::c_void, uriprop: Uri_PROPERTY, pbstrproperty: *mut ::std::mem::MaybeUninit<::windows_core::BSTR>, dwflags: u32) -> ::windows_core::HRESULT,
    pub GetPropertyLength: unsafe extern "system" fn(this: *mut ::core::ffi::c_void, uriprop: Uri_PROPERTY, pcchproperty: *mut u32, dwflags: u32) -> ::windows_core::HRESULT,
    pub GetPropertyDWORD: unsafe extern "system" fn(this: *mut ::core::ffi::c_void, uriprop: Uri_PROPERTY, pdwproperty: *mut u32, dwflags: u32) -> ::windows_core::HRESULT,
    #[cfg(feature = "Win32_Foundation")]
    ...
}
```
为IUri实现Com的接口
```rust
unsafe impl ::windows_core::Interface for IUri {
    type Vtable = IUri_Vtbl;
}
unsafe impl ::windows_core::ComInterface for IUri {
    const IID: ::windows_core::GUID = ::windows_core::GUID::from_u128(0xa39ee748_6a27_4817_a6f2_13914bef5890);
}
```
这时，我们就可以添加对`IUri_Vtbl`方法表的调用了。

```rust

impl IUri {
    pub unsafe fn GetDomain(&self) -> ::windows_core::Result<::windows_core::BSTR> {
        let mut result__ = ::std::mem::zeroed();
        (::windows_core::Interface::vtable(self).GetDomain)(::windows_core::Interface::as_raw(self), &mut result__).from_abi(result__)
    }
    pub unsafe fn GetPropertyLength(&self, uriprop: Uri_PROPERTY, pcchproperty: *mut u32, dwflags: u32) -> ::windows_core::Result<()> {
        (::windows_core::Interface::vtable(self).GetPropertyLength)(::windows_core::Interface::as_raw(self), uriprop, pcchproperty, dwflags).ok()
    }
}

```
我们的具体例子如下：

```rust
use windows::core::*;
use windows::Win32::System::Com::*;

fn main() -> windows::core::Result<()> {
    unsafe {
        let uri = CreateUri(w!("http://kennykerr.ca"), URI_CREATE_FLAGS::default(), 0)?;

        let domain = uri.GetDomain()?;
        let port = uri.GetPort()?;

        println!("{domain} ({port})");
        Ok(())
    }
}
```

## 间接包含IUnknown

```rust
pub struct IBitmapFrame(::windows_core::IUnknown);

#[repr(C)]
#[doc(hidden)]
pub struct IBitmapFrame_Vtbl {
    pub base__: ::windows_core::IInspectable_Vtbl,
    #[cfg(all(feature = "Foundation", feature = "Storage_Streams"))]
    pub GetThumbnailAsync: unsafe extern "system" fn(this: *mut ::core::ffi::c_void, result__: *mut *mut ::core::ffi::c_void) -> ::windows_core::HRESULT,
    #[cfg(not(all(feature = "Foundation", feature = "Storage_Streams")))]
    GetThumbnailAsync: usize,
    pub BitmapProperties: unsafe extern "system" fn(this: *mut ::core::ffi::c_void, result__: *mut *mut ::core::ffi::c_void) -> ::windows_core::HRESULT,
    pub BitmapPixelFormat: unsafe extern "system" fn(this: *mut ::core::ffi::c_void, result__: *mut BitmapPixelFormat) -> ::windows_core::HRESULT,
    pub BitmapAlphaMode: unsafe extern "system" fn(this: *mut ::core::ffi::c_void, result__: *mut BitmapAlphaMode) -> ::windows_core::HRESULT,
    pub DpiX: unsafe extern "system" fn(this: *mut ::core::ffi::c_void, result__: *mut f64) -> ::windows_core::HRESULT,
}

#[doc(hidden)]
#[repr(C)]
pub struct IInspectable_Vtbl {
    pub base: IUnknown_Vtbl,
    pub GetIids: unsafe extern "system" fn(this: *mut std::ffi::c_void, count: *mut u32, values: *mut *mut GUID) -> HRESULT,
    pub GetRuntimeClassName: unsafe extern "system" fn(this: *mut std::ffi::c_void, value: *mut *mut std::ffi::c_void) -> HRESULT,
    pub GetTrustLevel: unsafe extern "system" fn(this: *mut std::ffi::c_void, value: *mut i32) -> HRESULT,
}

#[doc(hidden)]
#[repr(C)]
pub struct IUnknown_Vtbl {
    pub QueryInterface: unsafe extern "system" fn(this: *mut std::ffi::c_void, iid: *const GUID, interface: *mut *mut std::ffi::c_void) -> HRESULT,
    pub AddRef: unsafe extern "system" fn(this: *mut std::ffi::c_void) -> u32,
    pub Release: unsafe extern "system" fn(this: *mut std::ffi::c_void) -> u32,
}

```
可以看到，还是使用的`IUnkonwn`做包含，同时设置了`base__`为`IInspectable_Vtbl`,而`IInspectable_Vtbl`包含了`IUnknown_Vtbl`

这里需要注意的`#[repr(C)]`可以保证与com本身的内存分布一致，而不是使用根据rust自身分布，保证了调用的一致性


## 多重继承

我们看到单一继承，通过base的包含方法获取到了对应的方法表。那么多重继承呢？




