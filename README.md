# Blog
*Notes on software practices and principles*

## Introduction
The following are my thoughts on coding practices that can help improve the reliability of our code. These practicies are necessarily more vague than standard coding practices, *like seperation of concerns*, because they're really about general ways to think about your code. With that in mind, I'll start with *The Nitpicking Principle.* (I'm looking for a better name for this. Please feel free to offer suggestions.)

## The Nitpicking Principle
This arose out of an observation I made a few years ago about software reliability: The difference between unreliable code and reliable code is often the difference between *usually* and *always*.

Here's what I mean. When I'm maintaining a project, many of the bugs that come in are problems that clearly need to be fixed, but some of them are edge cases. Unusual data that wasn't anticipated gets handled poorly. The software was written to handle the common cases, and ignored the uncommon ones. So their code would *usually* work. The irony is that usually it needed very minor rewrites to handle the edge cases well. Every poorly-handled edge that causes a bug will generate an issue report, which will pass through several hands as it moves through the bureaucracy, costing your company money. And these edge cases are often very easy to fix once they've been identified. So easy that it would have been worth it for the original developer to aim for *always works* rather than *usually works*.

Allow me to give you a fascinating example of this principle in action. To do so, I'll have to leave the field of software and take you to the field of medicine, where the stakes are much higher than increased maintenance costs. The example comes from what some doctors learned about treating cystic fibrosis. This example is drawn from **The Bell Curve**, an article by Dr. Atul Gawande, in the December 6, 2004 issue of *The New Yorker*, which is available on-line.

The article describes a certain health clinic for treating cystic fibrosis. Patients with this genetic disease suffer from excess lung congestion, and usually don't live past the age of 30. But this clinic has a much better success rate than other clinics, and the secret of their success wasn't about better drugs or better diagnosis, it was mostly due to the extraordinary effort they put into encouraging their patients into taking their medicine every single day, even when they were feeling healthy. Even when they were feeling great. Here's why. A healthy patient's chance of having a good day without medicine is 99.5%. That sounds pretty good. But with the medicine, it goes up to 99.95%. That seems like an insignificant difference. It hardly seems worth it. But that turns out to be a huge difference. There are 365 days in a year. When you multiply those odds through an entire year, the 99.95% comes to an 83% chance of making it through the year, while the 99.5% comes to only 16%. For cystic fibrosis patients, this can be the difference between living to be 30 versus living to be 45. If the patient has a child at age 20, this is the difference between dying when the child is 10 or 25. Some of their patients are reaching their 60s, which used to be unheard of for cystic fibrosis. So the medical staff at this clinic aims for the 99.95%. They aim for higher if there's a way. 99.5% is simply not good enough.

To get back to software, should we aim to write code that will usually work? It sounds reasonable, but it's often not a good idea. Many times I've seen code that will usually work, but a few trivial changes will turn it into code that will *always* work. So the coder didn't saving much time by settling for *usually*.

To apply this thinking to your application, think of it as a huge assembly of many small pieces of code. If all of them *usually* work, then the overall reliability might not be very good at all. If you have 10000 elements, each of which has a 1 in a thousand chance of failing on any given day, then you'll actually see ten failures a day. So aiming for *usually works* instead of *always works* is like aiming for 99.5% instead of 99.95%. So *usually* shouldn't be good enough, especially when *always* means minor changes to just a few lines of code. Of course, this is a simplification. But the point is that aiming for 100% reliability is well worth the effort.

How can we distinguish between *usually* and *always*? One way is to be aware of what I call the *Unnecessary Assumption Fallacy*.

## The Unnecessary Assumption Fallacy

Never make an assumption that you don't need to make. In writing software, we often make simplifying assumptions. If these assumptions are *always* correct, we're fine, but if they're not, we have inadvertently introduced a bug into our code. Even if they're *usually* correct. And in my experience, many of the assumptions we make are completely unnecessary. Never make an assumption that you don't need to make.

Here's an example.

Here's a method to find the youngest and oldest people in a database. The input list stores everyone's age.

    private static Range determineRange(List<Integer> ages) {
      int ageMin = 110;
      int ageMax = 0;
    
      for (int i: ages) {
        if (i < ageMin) {
          ageMin = i;
        }
        if (i > ageMax) {
          ageMax = i;
        }
      }
      return new Range(ageMin, ageMax);
    }
    
Since we're looking at people's ages, we can reasonably set a lower limit of zero and an upper limit of about 110, an age very few people reach. So we set those as our limits. This is fine as long as our assumption is correct.

But this method has two problems. The first one is that it assumes the array is not empty. If it gets an empty array, it returns a range with a minimum age of 110 and a maximum of zero! An empty array should probably cause an exception. The second problem is that we assume ages will range from 0 to 110.

Do we really know this? Let's put aside the fact that there are very rare records of people living beyond 110. The real problem is that we're assuming our data is valid. There could be a bug in the code that calculates people's ages, and we could get wildly unreasonable values. This method to determine ranges is incapable of alerting us to this bug. Do we really need to make any assumptions about the data? Instead of setting ageMin and ageMax to 110 and 0, why not set them to Integer.MAX_VALUE and Integer.MIN_VALUE?

This will draw outrageous but silly objections from other developers. They'll think you're assuming that people can live beyond a million years old. But you're not making that assumption. You're not assuming anything. You're not even assuming the data is valid. And more important, you now have a more generic method that will work for any kind of Integer data. The method will work for for file sizes as easily as people's ages. However, a better approach would be to initialize min and max to the first element of the collection, which is less likely to draw outrage from other developers.

The point is that, with a minor change, you can abandon your unnecessary assumption, and your code can't fail. If your data is invalid, you'll still get a correct range, so you'll know. The assumptions we made about people's ages offer us no advantage. And yeah, those assumptions are probably right, and the data will nearly always be valid. But do we gain anything by making those assumptions? 

Also, the change that takes us from *usually* works to *always* works was effortless. And yes, I've seen this issue come up in an actual project, where birth years were assumed to have 4 digits, but were being entered with 2 digits, so everyone was about 2000 years old. *Never make an assumption that you don't need to make.*

Here's a better version of the method that makes no assumptions, and can work with any data set. It will calculate the range of file sizes as easily as people's ages.

    private static Range determineRange(Iterable<Integer> data) {
      Iterator<Integer> iterator = data.iterator();
      int min = iterator.next();       // Throws an exception if data is empty.
      int max = min;
    
      while (iterator.hasNext()) {
        int i = iterator.next();
        if (i < min) {
          min = i;
        } else if (i > max) {
          max = i;
        }
      }
      return new Range(min, max);
    }
    
What assumptions am I no longer making?
* I'm not assuming the data represents people's ages.
* I'm not assuming the data is in a List structure.
* I'm not assuming the data is valid.
* I'm not assuming the list even has data.

Here we have a more robust method. Now this method can be used with any dataset. 

How useful is this change, when we're very unlikely to see this method fail? Think back to the example from the health clinic that aims for 99.95% instead of just 99.5%. By itself, this change really doesn't help reliability much. But it also doesn't cost much. It's marginally more reliable. But if all your code is written to be marginally more reliable, the cumulative effect can become very noticiable.

# Caveat

This principle should never be used to hide bugs.

Here's an example. Some developers take the view that you should ensure your code will never throw an exception in production. So many methods are written like this:

    public void processWidgetData(Widget widget) {
      if (widget != null) {
        ... // process the data
      }
    }

At first, it looks like this code is conforming to the nitpicking principle. They're catching a bug that's very easy to catch, to ensure that bad input won't cause an exception to be thrown. But is that what happens? Actually, if `widget` is null, the method won't work, regardless of whether you test it for null or not. You won't get an exception, but you also won't prevent a bug. You certainly won't make the method more reliable. All you will accomplish is to hide the bug. The exception that would get thrown without this test has a purpose. It notifies you that there's a bug in your code somewhere.

The Nitpicking Principle encourages you to write code that will always behave properly. But *the proper behavior of a method that gets invalid data structures is to throw an exception.* 

The `determineRange()` example above will throw an exception if the supplied `Iterable` contains no data. This is intentional. And it's reasonable, since there's no Range that makes sense for list with no data. 
