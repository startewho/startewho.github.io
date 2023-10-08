通过自己定义一个Com接口的形式了，来观察windows-rs的实现原理


我们定义了一个

```rs
se windows::{core::*, Win32::Foundation::*, Win32::System::Com::*};

// The `interface` macro defines a new local interface (based on IPersistMemory) that derives from an existing interface defined by the `windows` crate.
#[interface("BD1AE5E0-A6AE-11CE-BD37-504200C10000")]
unsafe trait ITestPersistMemory: IPersist {
    unsafe fn IsDirty(&self) -> HRESULT;
}

// The `implement` macro can implement both kinds of interfaces as the necessary type information is the same either way.
#[implement(ITestPersistMemory, IPersist)]
struct Test;
```

实现`ITestPersistMemory` 和`IPersist`接口
```rs
mpl IPersist_Impl for Test {
    fn GetClassID(&self) -> Result<GUID> {
        Ok("CEE1D356-0860-4262-90D4-C77423F0E352".into())
    }
}

impl ITestPersistMemory_Impl for Test {
    unsafe fn IsDirty(&self) -> HRESULT {
        S_FALSE
    }
}

```
注意：我们定义的是`IXXX`，实现trait使用的`IXXX_Impl`，是因为`#[interface]`他为我们的
```rs
[repr(transparent)]
struct ITestPersistMemory(IPersist);
impl ITestPersistMemory { unsafe fn IsDirty(&self ) -> HRESULT { (::windows::core::Interface::vtable(self).IsDirty)(::windows::core::Interface::as_raw(self) ) } }
unsafe impl ::windows::core::Interface for ITestPersistMemory { type Vtable = ITestPersistMemory_Vtbl; }
unsafe impl ::windows::core::ComInterface for ITestPersistMemory { const IID: ::windows::core::GUID = ::windows::core::GUID { data1: 0xBD1AE5E0, data2: 0xA6AE, data3: 0x11CE, data4: [0xBD, 0x37, 0x50, 0x42, 0x00, 0xC1, 0x00, 0x00] }; }
impl ::windows::core::RuntimeName for ITestPersistMemory {}
#[allow(non_camel_case_types)]
trait ITestPersistMemory_Impl: Sized + IPersist_Impl {
    unsafe fn IsDirty(&self ) -> HRESULT;
}
```
interace宏为我们实现`Interface`和`ComInterface`,`RuntimeName`
`ITestPersistMemory`则变成了struct

```rs
#[repr(C)]
struct Test_Impl<> where {
    identity: *const ::windows::core::IInspectable_Vtbl,
    vtables: (*const <ITestPersistMemory<> as ::windows::core::Interface>::Vtable, *const <IPersist<> as ::windows::core::Interface>::Vtable, ),
    this: Test::<>,
    count: ::windows::core::imp::WeakRefCount,
}
impl<> Test_Impl::<> where {
    const VTABLES: (<ITestPersistMemory<> as ::windows::core::Interface>::Vtable, <IPersist<> as ::windows::core::Interface>::Vtable, ) = (<ITestPersistMemory<> as ::windows::core::Interface>::Vtable::new::<Self, Test::<>, -1>(), <IPersist<> as ::windows::core::Interface>::Vtable::new::<Self, Test::<>, -2>(), );
    const IDENTITY: ::windows::core::IInspectable_Vtbl = ::windows::core::IInspectable_Vtbl::new::<Self, ITestPersistMemory<>, 0>();
    fn new(this: Test::<>) -> Self { Self { identity: &Self::IDENTITY, vtables: (&Self::VTABLES.0, &Self::VTABLES.1, ), this, count: ::windows::core::imp::WeakRefCount::new() } }
}
impl<> ::windows::core::IUnknownImpl for Test_Impl::<> where {
    type Impl = Test::<>;
    fn get_impl(&self) -> &Self::Impl { &self.this }
    unsafe fn QueryInterface(&self, iid: *const ::windows::core::GUID, interface: *mut *mut ::core::ffi::c_void) -> ::windows::core::HRESULT {
        if iid.is_null() || interface.is_null() { return ::windows::core::HRESULT(-2147467261); }
        *interface = if *iid == <::windows::core::IUnknown as ::windows::core::ComInterface>::IID || *iid == <::windows::core::IInspectable as ::windows::core::ComInterface>::IID || *iid == <::windows::core::imp::IAgileObject as ::windows::core::ComInterface>::IID { &self.identity as *const _ as *mut _ } else if <ITestPersistMemory<> as ::windows::core::Interface>::Vtable::matches(iid) { &self.vtables.0 as *const _ as *mut _ } else if <IPersist<> as ::windows::core::Interface>::Vtable::matches(iid) { &self.vtables.1 as *const _ as *mut _ } else { ::core::ptr::null_mut() };
        if !(*interface).is_null() {
            self.count.add_ref();
            return ::windows::core::HRESULT(0);
        }
        *interface = self.count.query(iid, &self.identity as *const _ as *mut _);
        if (*interface).is_null() { ::windows::core::HRESULT(-2147467262) } else { ::windows::core::HRESULT(0) }
    }
    fn AddRef(&self) -> u32 { self.count.add_ref() }
    unsafe fn Release(&self) -> u32 {
        let remaining = self.count.release();
        if remaining == 0 { _ = ::std::boxed::Box::from_raw(self as *const Self as *mut Self); }
        remaining
    }
}
impl<> Test::<> where {
    #[doc = r" Try casting as the provided interface"]
    #[doc = r""]
    #[doc = r" # Safety"]
    #[doc = r""]
    #[doc = r" This function can only be safely called if `self` has been heap allocated and pinned using"]
    #[doc = r" the mechanisms provided by `implement` macro."]
    unsafe fn cast<I: ::windows::core::ComInterface>(&self) -> ::windows::core::Result<I> {
        let boxed = (self as *const _ as *const *mut ::core::ffi::c_void).sub(1 + 2) as *mut Test_Impl::<>;
        let mut result = None;
        <Test_Impl::<> as ::windows::core::IUnknownImpl>::QueryInterface(&*boxed, &I::IID, &mut result as *mut _ as _).and_some(result)
    }
}
impl<> ::core::convert::From<Test::<>> for ::windows::core::IUnknown where {
    fn from(this: Test::<>) -> Self {
        let this = Test_Impl::<>::new(this);
        let boxed = ::core::mem::ManuallyDrop::new(::std::boxed::Box::new(this));
        unsafe { ::core::mem::transmute(&boxed.identity) }
    }
}
impl<> ::core::convert::From<Test::<>> for ::windows::core::IInspectable where {
    fn from(this: Test::<>) -> Self {
        let this = Test_Impl::<>::new(this);
        let boxed = ::core::mem::ManuallyDrop::new(::std::boxed::Box::new(this));
        unsafe { ::core::mem::transmute(&boxed.identity) }
    }
}
impl<> ::core::convert::From<Test::<>> for ITestPersistMemory<> where {
    fn from(this: Test::<>) -> Self {
        let this = Test_Impl::<>::new(this);
        let mut this = ::core::mem::ManuallyDrop::new(::std::boxed::Box::new(this));
        let vtable_ptr = &this.vtables.0;
        unsafe { ::core::mem::transmute(vtable_ptr) }
    }
}
impl<> ::windows::core::AsImpl<Test::<>> for ITestPersistMemory<> where {
    unsafe fn as_impl(&self) -> &Test::<> {
        let this = ::windows::core::Interface::as_raw(self);
        let this = (this as *mut *mut ::core::ffi::c_void).sub(1 + 0) as *mut Test_Impl::<>;
        &(*this).this
    }
}
impl<> ::core::convert::From<Test::<>> for IPersist<> where {
    fn from(this: Test::<>) -> Self {
        let this = Test_Impl::<>::new(this);
        let mut this = ::core::mem::ManuallyDrop::new(::std::boxed::Box::new(this));
        let vtable_ptr = &this.vtables.1;
        unsafe { ::core::mem::transmute(vtable_ptr) }
    }
}
impl<> ::windows::core::AsImpl<Test::<>> for IPersist<> where {
    unsafe fn as_impl(&self) -> &Test::<> {
        let this = ::windows::core::Interface::as_raw(self);
        let this = (this as *mut *mut ::core::ffi::c_void).sub(1 + 1) as *mut Test_Impl::<>;
        &(*this).this
    }
}
struct Test;
```

Test_Impl<>包含一个Test实例this，通过rust的Form/AsImpl，我们可以Test,TestImpl,IPersist<>,ITestPersistMemory<>,IUnknown,IInspectable互相转换。

原理就是根据内存分布，计算固定的偏移来实现，通过偏移计算到我们对应的Test_Impl，再取对应的字段即可。


还有QueryInterface的实现，通过比对guid，可以知道获取对应的

