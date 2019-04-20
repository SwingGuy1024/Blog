# Exception Anti-Patterns #
## 1 Catch, Log, Return null ##

    public HtmlEmailBuilder getEmailBuilder( /* several parameters */ ) {
        HtmlEmailBuilder builder = new HtmlEmailBuilder();
        try {
            // ~150 lines of code deleted.

                return builder;
            }
        } catch (Exception e) {
            LOGGER.error("getEmailBuilder", e);
        }
        return null;
    }

I was asked to debug a NullPointerException, with a stack trace. But that exception didn't help, because it was caused by 
the null value returned by this method. Of course, a more useful stack trace was right there in the log, but whoever filed the
bug had no idea that the preceeding exception was the important one to log, so I got 


--------

	public static void main(String[] args) {
		new BadPractices().example();
	}
	
	private void example() {
		String s = getString();
		System.out.println("String is " + s);
		
		s = "Second String";
		try {
			s = getString();
		} catch (Exception e) {
			e.printStackTrace();
		}
		System.out.println("String is " + s);
	}
	
	private String getString() {
		String s = "First string";
		try {
			s = badMethod();
		} finally {
			return s;
		}
	}
	
	private String badMethod() throws IOException {
		throw new RuntimeException("bad exception");
	}
