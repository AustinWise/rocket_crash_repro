

The read is happening on this link in Rocket:

https://github.com/SergioBenitez/Rocket/blob/v0.5-rc/core/lib/src/trip_wire.rs#L50

Note that the `TripWire` struct [implements `Deref`](https://github.com/SergioBenitez/Rocket/blob/v0.5-rc/core/lib/src/trip_wire.rs#L25).
This returns a reference to the `State` struct stored within an `Arc`.

The segfault is happening on the `ldarb` instruction:

```
        if self.tripped.load(Ordering::Acquire) {
00007FF6F27F9050  ldr         x8,[x0]  
00007FF6F27F9054  mov         x22,x0  
00007FF6F27F9058  mov         x19,x0  
00007FF6F27F905C  add         x8,x8,#0x38  
> 00007FF6F27F9060  ldarb       w8,[x8]  
```

Which is an [Load-Acquire Register Byte](https://developer.arm.com/documentation/ddi0596/2021-12/Base-Instructions/LDARB--Load-Acquire-Register-Byte-?lang=en)
instruction. Register `X8` is `0x38`. Since the [ArcInner struct](https://github.com/rust-lang/rust/blob/1.58.1/library/alloc/src/sync.rs#L316-L324)
stores data inline, this implies that the [ptr field in Arc](https://github.com/rust-lang/rust/blob/1.58.1/library/alloc/src/sync.rs#L236)
was null.

## Environment

Note that the software is being built on a x64 System and run on an ARM64 VM.

|Software|Version|
|--|--|
|Rust|1.58.1|
|MSVC|14.30.30705|
|Windows Host (in VM)|Windows 11 (10.0.22000.434)|
|HyperVisor|Parallels|
|HyperVisor Host|M1 Mac Mini with macos 12.1|

See [this article](https://docs.microsoft.com/en-us/windows/win32/wer/collecting-user-mode-dumps)
for information about how to collect crash dumps on Windows.


## Stack Trace

```
>	rocket_crash_repro.exe!rocket::trip_wire::impl$3::poll(core::pin::Pin<ref_mut$<rocket::trip_wire::TripWire>> self, core::task::wake::Context * cx) Line 50	Unknown
 	rocket_crash_repro.exe!hyper::server::shutdown::impl$1::poll<rocket_http::listener::Incoming<rocket::ext::CancellableListener<rocket::shutdown::Shutdown,tokio::net::tcp::listener::TcpListener>>,rocket::ext::CancellableIo<rocket::shutdown::Shutdown,tokio::net::tcp::stream::TcpStream>,std::io::error::Error,hyper::service::make::MakeServiceFn<rocket::server::impl$0::http_server::generator$0::closure$0>,hyper::body::body::Body,rocket::shutdown::Shutdown,enum$<hyper::common::exec::Exec, 1, 18446744073709551615, Executor>>(core::pin::Pin<ref_mut$<hyper::server::shutdown::Graceful<rocket_http::listener::Incoming<rocket::ext::CancellableListener<rocket::shutdown::Shutdown,tokio::net::tcp::listener::TcpListener>>,hyper::service::make::MakeServiceFn<rocket::server::impl$0::http_server::generator$0::closure$0>,rocket::shutdown::Shutdown,enum$<hyper::common::exec::Exec, 1, 18446744073709551615, Executor>>>> self, core::task::wake::Context * cx) Line 74	Unknown
 	[Inline Frame] rocket_crash_repro.exe!futures_core::future::impl$2::try_poll(core::pin::Pin<ref_mut$<hyper::server::shutdown::Graceful<rocket_http::listener::Incoming<rocket::ext::CancellableListener<rocket::shutdown::Shutdown,tokio::net::tcp::listener::TcpListener>>,hyper::service::make::MakeServiceFn<rocket::server::impl$0::http_server::generator$0::closure$0>,rocket::shutdown::Shutdown,enum$<hyper::common::exec::Exec, 1, 18446744073709551615, Executor>>>> self, core::task::wake::Context * cx) Line 82	Unknown
 	[Inline Frame] rocket_crash_repro.exe!futures_util::future::try_future::into_future::impl$2::poll(core::pin::Pin<ref_mut$<futures_util::future::try_future::into_future::IntoFuture<hyper::server::shutdown::Graceful<rocket_http::listener::Incoming<rocket::ext::CancellableListener<rocket::shutdown::Shutdown,tokio::net::tcp::listener::TcpListener>>,hyper::service::make::MakeServiceFn<rocket::server::impl$0::http_server::generator$0::closure$0>,rocket::shutdown::Shutdown,enum$<hyper::common::exec::Exec, 1, 18446744073709551615, Executor>>>>> self, core::task::wake::Context * cx) Line 34	Unknown
 	rocket_crash_repro.exe!futures_util::future::future::map::impl$2::poll<futures_util::future::try_future::into_future::IntoFuture<hyper::server::shutdown::Graceful<rocket_http::listener::Incoming<rocket::ext::CancellableListener<rocket::shutdown::Shutdown,tokio::net::tcp::listener::TcpListener>>,hyper::service::make::MakeServiceFn<rocket::server::impl$0::http_server::generator$0::closure$0>,rocket::shutdown::Shutdown,enum$<hyper::common::exec::Exec, 1, 18446744073709551615, Executor>>>,futures_util::fns::MapErrFn<rocket::server::impl$0::http_server::generator$0::closure$1>,enum$<core::result::Result<tuple$<>,rocket::error::Error>, 0, 7, Err>>(core::pin::Pin<ref_mut$<enum$<futures_util::future::future::map::Map<futures_util::future::try_future::into_future::IntoFuture<hyper::server::shutdown::Graceful<rocket_http::listener::Incoming<rocket::ext::CancellableListener<rocket::shutdown::Shutdown,tokio::net::tcp::listener::TcpListener>>,hyper::service::make::MakeServiceFn<rocket::server::impl$0::http_server::generator$0::closure$0>,rocket::shutdown::Shutdown,enum$<hyper::common::exec::Exec, 1, 18446744073709551615, Executor>>>,futures_util::fns::MapErrFn<rocket::server::impl$0::http_server::generator$0::closure$1>>, 0, 1, Incomplete>>> self, core::task::wake::Context * cx) Line 55	Unknown
 	[Inline Frame] rocket_crash_repro.exe!futures_util::future::future::impl$15::poll(core::pin::Pin<ref_mut$<futures_util::future::future::Map<futures_util::future::try_future::into_future::IntoFuture<hyper::server::shutdown::Graceful<rocket_http::listener::Incoming<rocket::ext::CancellableListener<rocket::shutdown::Shutdown,tokio::net::tcp::listener::TcpListener>>,hyper::service::make::MakeServiceFn<rocket::server::impl$0::http_server::generator$0::closure$0>,rocket::shutdown::Shutdown,enum$<hyper::common::exec::Exec, 1, 18446744073709551615, Executor>>>,futures_util::fns::MapErrFn<rocket::server::impl$0::http_server::generator$0::closure$1>>>> self, core::task::wake::Context * cx) Line 91	Unknown
 	[Inline Frame] rocket_crash_repro.exe!futures_util::future::try_future::impl$61::poll(core::pin::Pin<ref_mut$<futures_util::future::try_future::MapErr<hyper::server::shutdown::Graceful<rocket_http::listener::Incoming<rocket::ext::CancellableListener<rocket::shutdown::Shutdown,tokio::net::tcp::listener::TcpListener>>,hyper::service::make::MakeServiceFn<rocket::server::impl$0::http_server::generator$0::closure$0>,rocket::shutdown::Shutdown,enum$<hyper::common::exec::Exec, 1, 18446744073709551615, Executor>>,rocket::server::impl$0::http_server::generator$0::closure$1>>> self, core::task::wake::Context * cx) Line 91	Unknown
 	[Inline Frame] rocket_crash_repro.exe!core::future::future::impl$1::poll(core::pin::Pin<ref_mut$<core::pin::Pin<ref_mut$<futures_util::future::try_future::MapErr<hyper::server::shutdown::Graceful<rocket_http::listener::Incoming<rocket::ext::CancellableListener<rocket::shutdown::Shutdown,tokio::net::tcp::listener::TcpListener>>,hyper::service::make::MakeServiceFn<rocket::server::impl$0::http_server::generator$0::closure$0>,rocket::shutdown::Shutdown,enum$<hyper::common::exec::Exec, 1, 18446744073709551615, Executor>>,rocket::server::impl$0::http_server::generator$0::closure$1>>>>> self, core::task::wake::Context * cx) Line 119	Unknown
 	[Inline Frame] rocket_crash_repro.exe!futures_util::future::future::FutureExt::poll_unpin(core::pin::Pin<ref_mut$<futures_util::future::try_future::MapErr<hyper::server::shutdown::Graceful<rocket_http::listener::Incoming<rocket::ext::CancellableListener<rocket::shutdown::Shutdown,tokio::net::tcp::listener::TcpListener>>,hyper::service::make::MakeServiceFn<rocket::server::impl$0::http_server::generator$0::closure$0>,rocket::shutdown::Shutdown,enum$<hyper::common::exec::Exec, 1, 18446744073709551615, Executor>>,rocket::server::impl$0::http_server::generator$0::closure$1>>> * self, core::task::wake::Context * cx) Line 562	Unknown
 	rocket_crash_repro.exe!futures_util::future::select::impl$1::poll<rocket::shutdown::Shutdown,core::pin::Pin<ref_mut$<futures_util::future::try_future::MapErr<hyper::server::shutdown::Graceful<rocket_http::listener::Incoming<rocket::ext::CancellableListener<rocket::shutdown::Shutdown,tokio::net::tcp::listener::TcpListener>>,hyper::service::make::MakeServiceFn<rocket::server::impl$0::http_server::generator$0::closure$0>,rocket::shutdown::Shutdown,enum$<hyper::common::exec::Exec, 1, 18446744073709551615, Executor>>,rocket::server::impl$0::http_server::generator$0::closure$1>>>>(core::pin::Pin<ref_mut$<futures_util::future::select::Select<rocket::shutdown::Shutdown,core::pin::Pin<ref_mut$<futures_util::future::try_future::MapErr<hyper::server::shutdown::Graceful<rocket_http::listener::Incoming<rocket::ext::CancellableListener<rocket::shutdown::Shutdown,tokio::net::tcp::listener::TcpListener>>,hyper::service::make::MakeServiceFn<rocket::server::impl$0::http_server::generator$0::closure$0>,rocket::shutdown::Shutdown,enum$<hyper::common::exec::Exec, 1, 18446744073709551615, Executor>>,rocket::server::impl$0::http_server::generator$0::closure$1>>>>>> self, core::task::wake::Context * cx) Line 105	Unknown
 	[Inline Frame] rocket_crash_repro.exe!core::future::from_generator::impl$1::poll(core::pin::Pin<ref_mut$<core::future::from_generator::GenFuture<rocket::server::impl$0::default_tcp_http_server::generator$0>>> self, core::task::wake::Context * cx) Line 80	Unknown
 	[Inline Frame] rocket_crash_repro.exe!rocket::rocket::impl$1::_launch::generator$0(core::pin::Pin<ref_mut$<rocket::rocket::impl$1::_launch::generator$0>>, core::future::ResumeTy) Line 619	Unknown
 	rocket_crash_repro.exe!core::future::from_generator::impl$1::poll<rocket::rocket::impl$1::_launch::generator$0>(core::pin::Pin<ref_mut$<core::future::from_generator::GenFuture<rocket::rocket::impl$1::_launch::generator$0>>> self, core::task::wake::Context * cx) Line 80	Unknown
 	[Inline Frame] rocket_crash_repro.exe!core::future::from_generator::impl$1::poll(core::pin::Pin<ref_mut$<core::future::from_generator::GenFuture<rocket::rocket::impl$3::launch::generator$0>>> self, core::task::wake::Context * cx) Line 80	Unknown
 	[Inline Frame] rocket_crash_repro.exe!rocket_crash_repro::main::generator$0(core::pin::Pin<ref_mut$<rocket_crash_repro::main::generator$0>>, core::future::ResumeTy) Line 10	Unknown
 	rocket_crash_repro.exe!core::future::from_generator::impl$1::poll<rocket_crash_repro::main::generator$0>(core::pin::Pin<ref_mut$<core::future::from_generator::GenFuture<rocket_crash_repro::main::generator$0>>> self, core::task::wake::Context * cx) Line 80	Unknown
 	[Inline Frame] rocket_crash_repro.exe!tokio::park::thread::impl$5::block_on::closure$0(tokio::park::thread::impl$5::block_on::closure$0) Line 263	Unknown
 	[Inline Frame] rocket_crash_repro.exe!tokio::coop::with_budget::closure$0(tokio::coop::with_budget::closure$0 cell, core::cell::Cell<tokio::coop::Budget> *) Line 102	Unknown
 	[Inline Frame] rocket_crash_repro.exe!std::thread::local::LocalKey<core::cell::Cell<tokio::coop::Budget>>::try_with(tokio::coop::with_budget::closure$0 self) Line 399	Unknown
 	[Inline Frame] rocket_crash_repro.exe!std::thread::local::LocalKey<core::cell::Cell<tokio::coop::Budget>>::with(tokio::coop::with_budget::closure$0 self) Line 375	Unknown
 	[Inline Frame] rocket_crash_repro.exe!tokio::coop::with_budget(tokio::coop::Budget) Line 95	Unknown
 	[Inline Frame] rocket_crash_repro.exe!tokio::coop::budget(tokio::park::thread::impl$5::block_on::closure$0) Line 72	Unknown
 	rocket_crash_repro.exe!tokio::park::thread::CachedParkThread::block_on<core::future::from_generator::GenFuture<rocket_crash_repro::main::generator$0>>(core::future::from_generator::GenFuture<rocket_crash_repro::main::generator$0> self) Line 263	Unknown
 	[Inline Frame] rocket_crash_repro.exe!tokio::runtime::enter::Enter::block_on(core::future::from_generator::GenFuture<rocket_crash_repro::main::generator$0> self) Line 151	Unknown
 	rocket_crash_repro.exe!tokio::runtime::thread_pool::ThreadPool::block_on<core::future::from_generator::GenFuture<rocket_crash_repro::main::generator$0>>(core::future::from_generator::GenFuture<rocket_crash_repro::main::generator$0> self) Line 77	Unknown
 	rocket_crash_repro.exe!tokio::runtime::Runtime::block_on<core::future::from_generator::GenFuture<rocket_crash_repro::main::generator$0>>(core::future::from_generator::GenFuture<rocket_crash_repro::main::generator$0> self, core::panic::location::Location * future) Line 463	Unknown
 	[Inline Frame] rocket_crash_repro.exe!rocket::async_main(core::future::from_generator::GenFuture<rocket_crash_repro::main::generator$0> fut) Line 226	Unknown
 	rocket_crash_repro.exe!rocket_crash_repro::main() Line 10	Unknown
 	[Inline Frame] rocket_crash_repro.exe!core::ops::function::FnOnce::call_once(void(*)()) Line 227	Unknown
 	rocket_crash_repro.exe!std::sys_common::backtrace::__rust_begin_short_backtrace<void (*)(),tuple$<>>(void(*)() f) Line 129	Unknown
 	rocket_crash_repro.exe!std::rt::lang_start::closure$0<tuple$<>>(std::rt::lang_start::closure$0 *) Line 145	Unknown
 	[Inline Frame] rocket_crash_repro.exe!core::ops::function::impls::impl$2::call_once() Line 259	Unknown
 	[Inline Frame] rocket_crash_repro.exe!std::panicking::try::do_call() Line 406	Unknown
 	[Inline Frame] rocket_crash_repro.exe!std::panicking::try() Line 370	Unknown
 	[Inline Frame] rocket_crash_repro.exe!std::panic::catch_unwind() Line 133	Unknown
 	[Inline Frame] rocket_crash_repro.exe!std::rt::lang_start_internal::closure$2() Line 128	Unknown
 	[Inline Frame] rocket_crash_repro.exe!std::panicking::try::do_call() Line 406	Unknown
 	[Inline Frame] rocket_crash_repro.exe!std::panicking::try() Line 370	Unknown
 	[Inline Frame] rocket_crash_repro.exe!std::panic::catch_unwind() Line 133	Unknown
 	rocket_crash_repro.exe!std::rt::lang_start_internal() Line 128	Unknown
 	rocket_crash_repro.exe!main()	Unknown
 	[Inline Frame] rocket_crash_repro.exe!invoke_main() Line 78	C++
 	rocket_crash_repro.exe!__scrt_common_main_seh() Line 288	C++
 	[Inline Frame] rocket_crash_repro.exe!__scrt_common_main() Line 330	C++
 	rocket_crash_repro.exe!mainCRTStartup(void * __formal) Line 16	C++
 	kernel32.dll!BaseThreadInitThunk()	Unknown
 	ntdll.dll!RtlUserThreadStart()	Unknown
```