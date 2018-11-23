# Swagger Code Generation Options
## Spring
### Use Optional
This applies to RESTful APIs, but not to data models. If a RESTful API has an optional parameter, it wraps it in Optional. If a generated model has an optional field, its setter does not take Optional, and its getter does not return Optional.
*Example:*

    public ResponseEntity<Void> deletePet(
        @ApiParam(value = "Pet id to delete",required=true) @PathVariable("petId") Long petId,
        @ApiParam(value = "" ) @RequestHeader(value="api_key", required=false) Optional<String> apiKey
    ) {
        String accept = request.getHeader("Accept");
        return new ResponseEntity<Void>(HttpStatus.NOT_IMPLEMENTED);
    }

*Advise:* Leave this off.
### Interface Only
In the `api` folder, this generates only interfaces for the APIs. It leaves out the Controller classes, as well as these classes:

    ApiException.java
    ApiOriginFilter.java
    ApiResponseMessage.java
    NotFoundException.java
*Advise:* Turn this on after generating the controllers the first time.
