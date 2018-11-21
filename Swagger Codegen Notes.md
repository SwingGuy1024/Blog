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
