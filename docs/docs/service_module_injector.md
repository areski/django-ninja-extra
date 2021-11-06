## Injector

**Django Ninja Extra** APIController comes with dependency injector support.
Dependencies to a APIController can be resolved at constructor level automatically.
This is possible with python [**Injector** ](https://injector.readthedocs.io/en/latest/) library. 
Also, APIController injector works well with [**django_injector**](https://github.com/blubber/django_injector)

```python
from ninja import File
from ninja.files import UploadedFile
from ninja_extra import NinjaExtraAPI, APIController, route, router

class BucketFileUploadService:
    def upload_file_to_s3(self, file, bucket_name=None, acl="public-read", file_key=None):
        pass

    def upload_existing_file_to_s3(
            self, filepath, file_key, bucket_name=None, acl="public-read", delete_file_afterwards=False,
            clean_up_root_limit=None
    ):
        pass


@router('/user_profile')
class UserProfileController(APIController):
    def __init__(self, upload_service: BucketFileUploadService):
        self.upload_service = upload_service
    
    @route.post('/upload')
    def upload_profile_pic(self, file: UploadedFile = File(...)):
        self.upload_service.upload_file_to_s3(file=file)
        return {'message', 'uploaded successfully'}

    
api = NinjaExtraAPI(title='Injector Test')
api.register_controllers(UserProfileController)
```

## Module Registration
You can also register an injector module. And Inject the service to the APIController constructor
```python
import logging
import os

from ninja import File
from ninja.files import UploadedFile
from ninja_extra import NinjaExtraAPI, APIController, route, router

from typing import Any, cast
from django.conf import Settings
from injector import inject, Module, Binder, singleton

logger = logging.getLogger()


class BucketFileUpload:
    def upload_file_to_s3(self, file, bucket_name=None, acl="public-read", file_key=None):
        raise NotImplementedError()

    def upload_existing_file_to_s3(
            self, filepath, file_key, bucket_name=None, acl="public-read", delete_file_afterwards=False,
            clean_up_root_limit=None
    ):
        raise NotImplementedError()
    

class InMemoryBucketFileUpload(BucketFileUpload):
    @inject
    def __init__(self, settings: Settings):
        logger.info(f"===== Using InMemoryBucketFileUpload =======")
        self.settings = cast(Any, settings)

    def upload_file_to_s3(self, file, bucket_name=None, acl="public-read", file_key=None):
        logger.info(
            f"InMemoryBucketFileUpload ---- "
            f"upload_file_to_s3(file={file.filename}, bucket_name{bucket_name}, acl={acl}, file_key={file_key})"
        )
        if not file_key:
            return os.path.join(self.settings.UPLOAD_FOLDER, file.filename)
        return os.path.join(self.settings.BASE_DIR, file_key)

    def upload_existing_file_to_s3(self, filepath, file_key, bucket_name=None, acl="public-read",
                                   delete_file_afterwards=False, clean_up_root_limit=None):
        logger.info(f"InMemoryBucketFileUpload ---- upload_existing_file_to_s3("
                    f"filepath={filepath}, file_key={file_key}, "
                    f"bucket_name={bucket_name}, acl={acl}, delete_file_afterwards={delete_file_afterwards})")
        return filepath

    
class FileServiceModule(Module):
    def configure(self, binder: Binder) -> None:
        binder.bind(BucketFileUpload, to=InMemoryBucketFileUpload, scope=singleton)



@router('/user_profile')
class UserProfileController(APIController):
    def __init__(self, upload_service: BucketFileUpload):
        self.upload_service = upload_service
    
    @route.post('/upload')
    def upload_profile_pic(self, file: UploadedFile = File(...)):
        self.upload_service.upload_file_to_s3(file=file)
        assert isinstance(self.upload_service, InMemoryBucketFileUpload) # True
        return {'message', 'uploaded successfully'}

    
api = NinjaExtraAPI(title='Injector Test')
api.register_controllers(UserProfileController)
api.register_injector_modules(FileServiceModule)
```