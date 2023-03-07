# Study of services wrt realtime-compliance

## Sending service requests

`client::async_send_request(request)`</br>
`client::async_send_request(SharedRequest request, CallbackT && cb)`</br>
`client::async_send_request(SharedRequest request, CallbackT && cb)`</br>
- Call `async_send_request_impl()`, wich:
- Does `pending_requests_.try_emplace()`
  - With `pending_requests_` a `std::unordered_map`</br>
   <span style="color:red">**--> potential memory allocation – not realtime-compliant**</span> 

</br>
</br>


## Handling service requests

I assume that service requests are ultimately handled by a call to `Service::handle_request()`.
This calls `AnyServiceCallback::dispatch()`, followed by `Service::send_reponse()`.

</br>

### `AnyServiceCallback::dispatch()`

`AnyServiceCallback` emplaces callback functions in an `std::variant`, on call of `AnyServiceCallback::set()`.</br>
<span style="color:red">**According to** [this post](https://stackoverflow.com/questions/47876022/is-stdvariant-allowed-to-allocate-memory-for-its-members) **an std::variant does not allocate memory on emplace --> the use of an std::variant itself is realtime-compliant.**</span>

The variant is either of following types of callback function (or `std::monostate` if uninitialized),
that return `void` and take following arguments (as an `std::shared_ptr`):

1. `std::monostate`:  this is the 'empty' state  (i.e. not initialized)</br>
  </br>

2. `SharedPtrCallback`, with arguments:
   - `ServiceT::Request`
   - `ServiceT::Response`</br>
  </br>
3. `SharedPtrWithRequestHeaderCallback`, with arguments:
   - `rmw_request_id_t`
   - `ServiceT::Request`
   - `ServiceT::Response`</br>
   </br>

4. `SharedPtrDeferResponseCallback`, with arguments:
   - `rmw_request_id_t`
   - `ServiceT::Request`</br>
   </br>

5. `SharedPtrDeferResponseCallbackWithServiceHandle`, with arguments:
   - `rmw_request_id_t`
   - `rclcpp::Service<ServiceT>`
   - `ServiceT::Request`

</br>

The 'deferring' functions have an extra callback instead of the response, see following issues/PR:
- [Service server: Extend API to be able to defer a response #1707](https://github.com/ros2/rclcpp/issues/1707)
- [Add the ability to have asynchronous service callbacks #491](https://github.com/ros2/rclcpp/issues/491)
- [Corresponding pull request](https://github.com/ros2/rclcpp/pull/1709)

</br>
</br>


`AnyServiceCallback::dispatch()`

- In case of a `SharedPtrDeferResponseCallback` or `SharedPtrDeferResponseCallbackWithServiceHandle`:
  - Call the callback and return `nullptr`,
- In case of a `SharedPtrCallback` or `SharedPtrWithRequestHeaderCallback`:
  - Allocate a response,
  - Call the callback,
  - Return the response.</br>
  <span style="color:red">**Heap allocation, no support for a custom allocator -- not realtime-compliant**</span>

</br>

### `Service::send_response()`

This calls `rcl_send_response()` which, according to the code documentation, does not allocate memory and is lock-free.

</br>
</br>
