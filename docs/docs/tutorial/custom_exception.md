## Custom Exception

If you are coming from DRF, then you are use to `APIException` class.
**Django-Ninja-Extra** has something similar with the same and a has created
a `handler` for it.

```python
from ninja_extra.exceptions import APIException
from ninja_extra import status
from ninja_extra import router, APIController, route, NinjaExtraAPI


class CustomAPIException(APIException):
    status_code = status.HTTP_401_UNAUTHORIZED
    message = 'UnAuthorized'

    
@router('/users', tags=["exception"])
class MyController(APIController):
    @route.get('/exception')
    def custom_exception(self):
        raise CustomAPIException()

    
api = NinjaExtraAPI(title='Exception Test')
api.register_controllers(MyController)
```
![Preview](docs/images/custom_exception.gif)