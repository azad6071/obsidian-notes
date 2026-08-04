Mockito creates and injects mocks at runtime using reflection. Final fields must be initialized at construction time, so Mockito cannot assign them. Therefore, mocks should not be declared final.

when() -> use when method returns something
doThrow() -> when return type is void