# Swagger Notes:
## Data Types:
    string  int32     gives String
    integer int32     gives Integer
    number  float     gives Float
    number  double    gives Double
    number  int32     gives BigDecimal
    number  xxx       gives BigDecimal
    string  byte      gives byte[ ]
    string  binary    gives File
    string  date      gives LocalDate
    string  date-time gives OffsetDateTime
    string  xxx       gives String
## Code Generation Options
### Spring
#### Use Optional
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
#### Interface Only
In the `api` folder, this generates only interfaces for the APIs. It leaves out the Controller classes, as well as these classes:

    ApiException.java
    ApiOriginFilter.java
    ApiResponseMessage.java
    NotFoundException.java
*Advise:* Turn this on after generating the controllers the first time.
#### Use Bean Validation
I have no idea how to use this. It didn't make any changes to the generated code. All my efforts to modify the yaml file to force changes failed.
#### With XML
This added the following annotations to the model objects and its fields:

	@JacksonXmlRootElement(localName = "User")
	@XmlRootElement(name = "User")
	@XmlAccessorType(XmlAccessType.FIELD)
	public class User   {
	  @JsonProperty("id")
	  @JacksonXmlProperty(localName = "id")
	  private Long id = null;
	
	  @JsonProperty("username")
	  @JacksonXmlProperty(localName = "username")
	  private String username = null;
	  ...
I have no idea what most of these annotations do.
