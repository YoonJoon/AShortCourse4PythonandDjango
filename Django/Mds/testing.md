<!-- $theme: gaia -->

[Django Tutorial Part 10: Testing a Django web application](https://github.com/YoonJoon/AboutDjango/blob/master/testing.md)
=================================

<br>

##### Created by [이 윤 준](https://www.facebook.com/yoonjoon.lee) (yoonjoon.lee@gmail.com)

June, 2019

---

As websites grow they become harder to test manually. Not only is there more to test, but, as interactions between components become more complex, a small change in one area can impact other areas, so more changes will be required to ensure everything keeps working and errors are not introduced as more changes are made. 

One way to mitigate these problems is to write automated tests, which can easily and reliably be run every time we make a change. This shows how to automate <i>unit testing</i> of our website using Django's test framework.

---

### Overview

The <i>Local Library</i> currently has pages to display lists of all books and authors, detail views for <code>Book</code> and <code>Author</code> items, a page to renew <code>BookInstances</code>, and pages to create, update, and delete <code>Author</code> items and <code>Book</code>. 

Even with this relatively small site, manually navigating to each page and superficially checking that everything works as expected can take several minutes. 

As we make changes and grow the site, the time required to manually check that everything works "properly" will only grow. 

---

If we were to continue as we are, eventually we'd be spending most of our time testing, and very little time improving our code.

Automated tests can really help with this problem.

The obvious benefits are that they can be run much faster than manual tests, can test to a much lower level of detail, and test exactly the same functionality every time. 

Because they are fast, automated tests can be executed more regularly, and if a test fails, they point to exactly where code is not performing as expected.

---

In addition, automated tests can act as the first real-world "user" of our code, forcing us to be rigorous about defining and documenting how our website should behave. 

Often they are the basis for our code examples and documentation. For these reasons, some software development processes start with test definition and implementation, after which the code is written to match the required behavior.

This shows how to write automated tests for Django, by adding a number of tests to the <i>LocalLibrary</i> website.

---

#### Types of testing

- <b>Unit tests</b>: Verify functional behavior of individual components, often to class and function level.
- <b>Regression tests</b>: Tests that reproduce historic bugs. Each test is initially run to verify that the bug has been fixed, and then re-run to ensure that it has not been reintroduced following later changes to the code.
- <b>Integration tests</b>: Verify how groupings of components work when used together. Integration tests are aware of the required interactions between components, but not necessarily of the internal operations of each component. 

---

#### What does Django provide for testing?

Testing a website is a complex task, because it is made of several layers of logic – from HTTP-level request handling, queries models, to form validation and processing, and template rendering.

Django provides a test framework with a small hierarchy of classes that build on the Python standard unittest library. 

This test framework is suitable for both unit and integration tests. The Django framework adds API methods and tools to help test web and Django-specific behaviour. 

---

These allow us to simulate requests, insert test data, and inspect your application's output. 

Django also provides an API (LiveServerTestCase) and tools for using different testing frameworks, for example we can integrate with the popular Selenium framework to simulate a user interacting with a live browser.

---

To write a test we derive from any of the Django (or <i>unittest</i>) test base classes (SimpleTestCase, TransactionTestCase, TestCase, LiveServerTestCase) and then write separate methods to check that specific functionality works as expected (tests use "assert" methods to test that expressions result in <code>True</code> or <code>False</code> values, or that two values are equal, etc.).

When we start a test run, the framework executes the chosen test methods in our derived classes. The test methods are run independently, with common setup and/or tear-down behaviour defined in the class.

---

<font size="6">
  
```python
class YourTestClass(TestCase):
    def setUp(self):
        # Setup run before every test method.
        pass

    def tearDown(self):
        # Clean up run after every test method.
        pass

    def test_something_that_will_pass(self):
        self.assertFalse(False)

    def test_something_that_will_fail(self):
        self.assertTrue(False)
```
</font>

---

The best base class for most tests is <u>django.test.TestCase</u>. 

This test class creates a clean database before its tests are run, and runs every test function in its own transaction. 

The class also owns a test <u>Client</u> that we can use to simulate a user interacting with the code at the view level. 

We're going to concentrate on unit tests, created using this <u>TestCase</u> base class.

---

#### What should you test?

We should test all aspects of our own code, but not any libraries or functionality provided as part of Python or Django.


<font size="6">
  
```python
class Author(models.Model):
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)
    date_of_birth = models.DateField(null=True, blank=True)
    date_of_death = models.DateField('Died', null=True, blank=True)
    
    def get_absolute_url(self):
        return reverse('author-detail', args=[str(self.id)])
    
    def __str__(self):
        return '%s, %s' % (self.last_name, self.first_name)
```
</font>

---

So for example, consider the <code>Author</code> model. We don't need to explicitly test that <code>first_name</code> and <code>last_name</code> have been stored properly as <code>CharField</code> in the database because that is something defined by Django. Nor do we need to test that the <code>date_of_birth</code> has been validated to be a date field, because that is again something implemented in Django.

However we should check the text used for the labels (<i>First name</i>, <i>Last name</i>, <i>Date of birth</i>, <i>Died</i>), and the size of the field allocated for the text (<i>100 chars</i>), because these are part of our design and something that could be broken/changed in future.

---

Similarly, we should check that the custom methods <code>get_absolute_url()</code> and <code>\_\_str\_\_()</code> behave as required because they are our code/business logic. In the case of <code>get_absolute_url()</code> we can trust that the Django <code>reverse()</code> method has been implemented properly, so what we're testing is that the associated view has actually been defined.

---

### Test structure overview

Before we go into the detail of "what to test", let's first briefly look at where and how tests are defined.

Django uses the unittest module’s <u>built-in test discovery</u>, which will discover tests under the current working directory in any file named with the pattern <b>test*.py</b>. Provided we name the files appropriately, we can use any structure we like. 

It's recommend that we create a module for our test code, and have separate files for models, views, forms, and any other types of code we need to test. For example:

---

<font size="6">
  
```bash
catalog/
  /tests/
    __init__.py
    test_models.py
    test_forms.py
    test_views.py
```
</font>

Create a file structure as shown above in your LocalLibrary project. The <b>\_\_init\_\_.py</b> should be an empty file. We can create the three test files by copying and renaming the skeleton test file <b>/catalog/tests.py</b>.

Open <b>/catalog/tests/test_models.py</b>. The file should import <code>django.test.TestCase</code>.

---

<font size="6">
  
```python
from django.test import TestCase

# Create your tests here.
```
</font>

Often we will add a test class for each model/view/form we want to test, with individual methods for testing specific functionality. 

In other cases we may wish to have a separate class for testing a specific use case, with individual test functions that test aspects of that use-case (for example, a class to test that a model field is properly validated, with functions to test each of the possible failure cases). 

---

Again, the structure is very much up to us, but it is best if we are consistent.

Add the test class to the bottom of the file. The class demonstrates how to construct a test case class by deriving from TestCase.

---

<font size="5">
  
```python
class YourTestClass(TestCase):
    @classmethod
    def setUpTestData(cls):
        print("setUpTestData: Run once " + 
          "to set up non-modified data for all class methods.")
        pass

    def setUp(self):
        print("setUp: Run once for every test method to setup clean data.")
        pass

    def test_false_is_false(self):
        print("Method: test_false_is_false.")
        self.assertFalse(False)

    def test_false_is_true(self):
        print("Method: test_false_is_true.")
        self.assertTrue(False)

    def test_one_plus_one_equals_two(self):
        print("Method: test_one_plus_one_equals_two.")
        self.assertEqual(1 + 1, 2)
```
</font>

---

The new class defines two methods that we can use for pre-test configuration (for example, to create any models or other objects we will need for the test):

- <code>setUpTestData()</code> is called once at the beginning of the test run for class-level setup. We'd use this to create objects that aren't going to be modified or changed in any of the test methods.
- <code>setUp()</code> is called before every test function to set up any objects that may be modified by the test (every test function will get a "fresh" version of these objects).

---

Below those we have a number of test methods, which use <code>Assert</code> functions to test whether conditions are true, false or equal (<code>AssertTrue</code>, <code>AssertFalse</code>, <code>AssertEqual</code>). 

If the condition does not evaluate as expected then the test will fail and report the error to your console.

The <code>AssertTrue</code>, <code>AssertFalse</code>, <code>AssertEqual</code> are standard assertions provided by <b>unittest</b>. 

There are other standard assertions in the framework, and also <u>Django-specific assertions</u> to test if a view redirects (<code>assertRedirects</code>), to test if a particular template has been used (<code>assertTemplateUsed</code>), etc.

---

### How to run the tests

The easiest way to run all the tests is to use the command:

<font size="6">
  
```bash
py -3 manage.py test
```
</font>

This will discover all files named with the pattern test*.py under the current directory and run all tests defined using appropriate base classes (here we have a number of test files, but only /catalog/tests/test_models.py currently contains any tests). 

By default the tests will individually report only on test failures, followed by a test summary.

---

Run the tests in the root directory of <i>LocalLibrary</i>. We should see an output like the one.

<font size="5">
  
```bash
> python3 manage.py test

Creating test database for alias 'default'...
setUpTestData: Run once to set up non-modified data for all class methods.
setUp: Run once for every test method to setup clean data.
Method: test_false_is_false.
setUp: Run once for every test method to setup clean data.
Method: test_false_is_true.
setUp: Run once for every test method to setup clean data.
Method: test_one_plus_one_equals_two.
.
======================================================================
FAIL: test_false_is_true (catalog.tests.tests_models.YourTestClass)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "D:\Github\django_tmp\library_w_t_2\locallibrary\catalog\tests\tests_models.py", line 22, in test_false_is_true
    self.assertTrue(False)
AssertionError: False is not true

----------------------------------------------------------------------
Ran 3 tests in 0.075s

FAILED (failures=1)
Destroying test database for alias 'default'...
```
</font>

---

We see that we had one test failure, and we can see exactly what function failed and why (this failure is expected, because <code>False</code> is not <code>True</code>!).

The text shown in bold(?) would not normally appear in the test output (this is generated by the <code>print()</code> functions in our tests). This shows how the <code>setUpTestData()</code> method is called once for the class and <code>setUp()</code> is called before each method.

---

#### Showing more test information

If we want to get more information about the test run we can change the verbosity. 

For example, to list the test successes as well as failures (and a whole bunch of information about how the testing database is set up) you can set the verbosity to "2":

<font size="6">
  
```bash
py -3 manage.py test --verbosity 2
```
</font>

The allowed verbosity levels are 0, 1, 2, and 3, with the default being "1".

---

#### Running specific tests

If we want to run a subset of our tests we can do so by specifying the full dot path to the package(s), module, <code>TestCase</code> subclass or method:

<font size="5">
  
```bash
# Run the specified module
py -3 manage.py test catalog.tests

# Run the specified module
py -3 manage.py test catalog.tests.test_models

# Run the specified class
py -3 manage.py test catalog.tests.test_models.YourTestClass

# Run the specified method
py -3 manage.py test catalog.tests.test_models.YourTestClass.test_one_plus_one_equals_two
```

</font>

---

### LocalLIbrary tests

We know how to run our tests and what sort of things we need to test, let's look at some practical examples.

---

#### Models

We should test anything that is part of our design or that is defined by code that we have written.

For example, consider the <code>Author</code> model. 

We should test the labels for all the fields, because even though we haven't explicitly specified most of them, we have a design that says what these values should be. 

If we don't test the values, then we don't know that the field labels have their intended values. 

---

Similarly while we trust that Django will create a field of the specified length, it is worthwhile to specify a test for this length to ensure that it was implemented as planned.

<font size="5">
  
```python
class Author(models.Model):
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)
    date_of_birth = models.DateField(null=True, blank=True)
    date_of_death = models.DateField('Died', null=True, blank=True)
    
    def get_absolute_url(self):
        return reverse('author-detail', args=[str(self.id)])
    
    def __str__(self):
        return f'{self.last_name}, {self.first_name}'
```

</font>

Open our <b>/catalog/tests/test_models.py</b>, and replace any existing code with the following test code for the Author model.

---

We'll see that we first import <code>TestCase</code> and derive our test class (<code>AuthorModelTest</code>) from it, using a descriptive name so we can easily identify any failing tests in the test output. 

We then call <code>setUpTestData()</code> to create an author object that we will use but not modify in any of the tests.

<font size="5">
  
```python
from django.test import TestCase

from catalog.models import Author

class AuthorModelTest(TestCase):
    @classmethod
    def setUpTestData(cls):
        # Set up non-modified objects used by all test methods
        Author.objects.create(first_name='Big', last_name='Bob')
```

---

```python
    def test_first_name_label(self):
        author = Author.objects.get(id=1)
        field_label = author._meta.get_field('first_name').verbose_name
        self.assertEquals(field_label, 'first name')

    def test_date_of_death_label(self):
        author=Author.objects.get(id=1)
        field_label = author._meta.get_field('date_of_death').verbose_name
        self.assertEquals(field_label, 'died')

    def test_first_name_max_length(self):
        author = Author.objects.get(id=1)
        max_length = author._meta.get_field('first_name').max_length
        self.assertEquals(max_length, 100)

    def test_object_name_is_last_name_comma_first_name(self):
        author = Author.objects.get(id=1)
        expected_object_name = f'{author.last_name}, {author.first_name}'
        self.assertEquals(expected_object_name, str(author))

    def test_get_absolute_url(self):
        author = Author.objects.get(id=1)
        # This will also fail if the urlconf is not defined.
        self.assertEquals(author.get_absolute_url(), '/catalog/author/1')
```

</font>

---

The field tests check that the values of the field labels (<code>verbose_name</code>) and that the size of the character fields are as expected. These methods all have descriptive names, and follow the same pattern:

<font size="5">
  
```python
    # Get an author object to test
    author = Author.objects.get(id=1)

    # Get the metadata for the required field and use it 
    # to query the required field data
    field_label = author._meta.get_field('first_name').verbose_name

    # Compare the value to the expected result
    self.assertEquals(field_label, 'first name')
```

</font>

---

The interesting things to note are:

<font size="6">

- We can't get the <code>verbose_name</code> directly using <code>author.first_name.verbose_name</code>, because <code>author.first_name</code> is a string (not a handle to the first_name object that we can use to access its properties). Instead we need to use the author's <code>_meta</code> attribute to get an instance of the field and use that to query for the additional information.
- We chose to use <code>assertEquals(field_label,'first name')</code> rather than <code>assertTrue(field_label == 'first name')</code>. The reason for this is that if the test fails the output for the former tells you what the label actually was, which makes debugging the problem just a little easier.

</font>

---

We also need to test our custom methods. These essentially just check that the object name was constructed as we expected using "Last Name", "First Name" format, and that the URL we get for an <code>Author</code> item is as we would expect.

<font size="5">
  
```python
    def test_object_name_is_last_name_comma_first_name(self):
        author = Author.objects.get(id=1)
        expected_object_name = f'{author.last_name}, {author.first_name}'
        self.assertEquals(expected_object_name, str(author))
    
    def test_get_absolute_url(self):
        author = Author.objects.get(id=1)
        # This will also fail if the urlconf is not defined.
        self.assertEquals(author.get_absolute_url(), '/catalog/author/1')
```

</font>

---

Run the tests. 

If you created the Author model as we described in the models tutorial it is quite likely that you will get an error for the <code>date_of_death</code> label. The test is failing because it was written expecting the label definition to follow Django's convention of not capitalising the first letter of the label (Django does this for us).

<font size="5">
  
```bash
======================================================================
FAIL: test_date_of_death_label (catalog.tests.test_models.AuthorModelTest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "D:\...\locallibrary\catalog\tests\test_models.py", line 32, in test_date_of_death_label
    self.assertEquals(field_label,'died')
AssertionError: 'Died' != 'died'
- Died
? ^
+ died
? ^
```

</font>

---

This is a very minor bug, but it does highlight how writing tests can more thoroughly check any assumptions you may have made.

The patterns for testing the other models are similar. Feel free to create your own tests for our other models.

---

#### Forms

The philosophy for testing our forms is the same as for testing our models; we need to test anything that we've coded or our design specifies, but not the behaviour of the underlying framework and other third party libraries.

Generally this means that you should test that the forms have the fields that you want, and that these are displayed with appropriate labels and help text. You don't need to verify that Django validates the field type correctly (unless you created your own custom field and validation) — i.e. you don't need to test that an email field only accepts emails. 

---

However you would need to test any additional validation that you expect to be performed on the fields and any messages that your code will generate for errors.

Consider our form for renewing books. This has just one field for the renewal date, which will have a label and help text that we will need to verify.

---

<font size="5">
  
```python
class RenewBookForm(forms.Form):
    """Form for a librarian to renew books."""
    renewal_date = forms.DateField(
        help_text="Enter a date between now and 4 weeks (default 3).")

    def clean_renewal_date(self):
        data = self.cleaned_data['renewal_date']

        # Check if a date is not in the past.
        if data < datetime.date.today():
            raise ValidationError(_('Invalid date - renewal in past'))

        # Check if date is in the allowed range (+4 weeks from today).
        if data > datetime.date.today() + datetime.timedelta(weeks=4):
            raise ValidationError(
                _('Invalid date - renewal more than 4 weeks ahead'))

        # Remember to always return the cleaned data.
        return data
```

</font>

---

Open our <b>/catalog/tests/test_forms.py</b> file and replace any existing code with the test code for the <code>RenewBookForm</code> form. 

We start by importing our form and some Python and Django libraries to help test time-related functionality. 

We then declare our form test class in the same way as we did for models, using a descriptive name for our <code>TestCase</code>-derived test class.

<font size="5">
  
```python
import datetime

from django.test import TestCase
from django.utils import timezone

from catalog.forms import RenewBookForm
```

---

```pyhon
class RenewBookFormTest(TestCase):
    def test_renew_form_date_field_label(self):
        form = RenewBookForm()
        self.assertTrue(form.fields['renewal_date'].label == None or form.fields['renewal_date'].label == 'renewal date')

    def test_renew_form_date_field_help_text(self):
        form = RenewBookForm()
        self.assertEqual(form.fields['renewal_date'].help_text, 
            'Enter a date between now and 4 weeks (default 3).')

    def test_renew_form_date_in_past(self):
        date = datetime.date.today() - datetime.timedelta(days=1)
        form = RenewBookForm(data={'renewal_date': date})
        self.assertFalse(form.is_valid())

    def test_renew_form_date_too_far_in_future(self):
        date = datetime.date.today() + datetime.timedelta(weeks=4) + 
            datetime.timedelta(days=1)
        form = RenewBookForm(data={'renewal_date': date})
        self.assertFalse(form.is_valid())

    def test_renew_form_date_today(self):
        date = datetime.date.today()
        form = RenewBookForm(data={'renewal_date': date})
        self.assertTrue(form.is_valid())
        
    def test_renew_form_date_max(self):
        date = timezone.now() + datetime.timedelta(weeks=4)
        form = RenewBookForm(data={'renewal_date': date})
        self.assertTrue(form.is_valid())
```

</font>

---

The first two functions test that the field's <code>label</code> and <code>help_text</code> are as expected. 

We have to access the field using the fields dictionary (e.g. <code>form.fields['renewal_date']</code>). 

We also have to test whether the label value is None, because even though Django will render the correct label it returns None if the value is not explicitly set.

The rest of the functions test that the form is valid for renewal dates just inside the acceptable range and invalid for values outside the range. 

---

Note how we construct test date values around our current date (<code>datetime.date.today()</code>) using <code>datetime.timedelta()</code> (in this case specifying a number of days or weeks). We then just create the form, passing in our data, and test if it is valid.

We do have some others, but they are automatically created by our generic class-based editing views, and should be tested there. 

Run the tests and confirm that our code still passes!

---

### Views

To validate our view behaviour we use the Django test <u>Client</u>. 

This class acts like a dummy web browser that we can use to simulate <code>GET</code> and <code>POST</code> requests on a URL and observe the response. 

We can see almost everything about the response, from low-level HTTP (result headers and status codes) through to the template we're using to render the HTML and the context data we're passing to it.

We can also see the chain of redirects (if any) and check the URL and status code at each step. 

---

This allows us to verify that each view is doing what is expected.

Let's start with one of our simplest views, which provides a list of all Authors. This is displayed at URL <b>/catalog/authors/</b> (an URL named 'authors' in the URL configuration).

<font size="5">
  
```pyhon
class AuthorListView(generic.ListView):
    model = Author
    paginate_by = 10
```

</font>

As this is a generic list view almost everything is done for us by Django. 

---

Arguably if you trust Django then the only thing you need to test is that the view is accessible at the correct URL and can be accessed using its name. 

However if you're using a test-driven development process you'll start by writing tests that confirm that the view displays all Authors, paginating them in lots of 10.

Open the <b>/catalog/tests/test_views.py</b> file and replace any existing text with the following test code for <code>AuthorListView</code>. As before we import our model and some useful classes. 

In the <code>setUpTestData()</code> method we set up a number of <code>Author</code> objects so that we can test our pagination.

---

<font size="5">
  
```python
from django.test import TestCase
from django.urls import reverse

from catalog.models import Author

class AuthorListViewTest(TestCase):
    @classmethod
    def setUpTestData(cls):
        # Create 13 authors for pagination tests
        number_of_authors = 13

        for author_id in range(number_of_authors):
            Author.objects.create(
                first_name=f'Christian {author_id}',
                last_name=f'Surname {author_id}',
            )
           
    def test_view_url_exists_at_desired_location(self):
        response = self.client.get('/catalog/authors/')
        self.assertEqual(response.status_code, 200)
           
    def test_view_url_accessible_by_name(self):
        response = self.client.get(reverse('authors'))
        self.assertEqual(response.status_code, 200)
        
    def test_view_uses_correct_template(self):
        response = self.client.get(reverse('authors'))
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, 'catalog/author_list.html')
```

---

```pyhon        
    def test_pagination_is_ten(self):
        response = self.client.get(reverse('authors'))
        self.assertEqual(response.status_code, 200)
        self.assertTrue('is_paginated' in response.context)
        self.assertTrue(response.context['is_paginated'] == True)
        self.assertTrue(len(response.context['author_list']) == 10)

    def test_lists_all_authors(self):
        # Get second page and confirm it has (exactly) remaining 3 items
        response = self.client.get(reverse('authors')+'?page=2')
        self.assertEqual(response.status_code, 200)
        self.assertTrue('is_paginated' in response.context)
        self.assertTrue(response.context['is_paginated'] == True)
        self.assertTrue(len(response.context['author_list']) == 3)
```

</font>

All the tests use the client (belonging to our <code>TestCase</code>'s derived class) to simulate a <code>GET</code> request and get a response. The first version checks a specific URL (note, just the specific path without the domain) while the second generates the URL from its name in the URL configuration.

---

<font size="5">
  
```python
    response = self.client.get('/catalog/authors/')
    response = self.client.get(reverse('authors'))
```

</font>

Once we have the response we query it for its status code, the template used, whether or not the response is paginated, the number of items returned, and the total number of items.

---

The most interesting variable we demonstrate is <code>response.context</code>, which is the context variable passed to the template by the view. 

This is incredibly useful for testing, because it allows us to confirm that our template is getting all the data it needs. 

In other words we can check that we're using the intended template and what data the template is getting, which goes a long way to verifying that any rendering issues are solely due to template.

---

##### Views that are restricted to logged in users

In some cases we'll want to test a view that is restricted to just logged in users. 

For example our <code>LoanedBooksByUserListView</code> is very similar to our previous view but is only available to logged in users, and only displays <code>BookInstance</code> records that are borrowed by the current user, have the 'on loan' status, and are ordered "oldest first".

---

<font size="5">
  
```python
from django.contrib.auth.mixins import LoginRequiredMixin

class LoanedBooksByUserListView(LoginRequiredMixin, generic.ListView):
    """Generic class-based view listing books on loan to current user."""
    model = BookInstance
    template_name ='catalog/bookinstance_list_borrowed_user.html'
    paginate_by = 10

    def get_queryset(self):
        return BookInstance.objects.filter(borrower=self.request.user).
            filter(status__exact='o').order_by('due_back')
```

</font>

Add the test code to <b>/catalog/tests/test_views.py</b>. 

Here we first use <code>SetUp()</code> to create some user login accounts and <code>BookInstance</code> objects (along with their associated books and other records) that we'll use later in the tests. 

---

Half of the books are borrowed by each test user, but we've initially set the status of all books to "maintenance".

We've used <code>SetUp()</code> rather than <code>setUpTestData()</code> because we'll be modifying some of these objects later.

<font size="5">
  
```python
import datetime

from django.utils import timezone
from django.contrib.auth.models import User # Required to assign User as a borrower

from catalog.models import BookInstance, Book, Genre, Language

class LoanedBookInstancesByUserListViewTest(TestCase):
    def setUp(self):
        # Create two users
        test_user1 = User.objects.create_user(username='testuser1', 
            password='1X<ISRUkw+tuK')
        test_user2 = User.objects.create_user(username='testuser2', 
            password='2HJ1vRV0Z&3iD')
```

---

```pyhon         
        test_user1.save()
        test_user2.save()
        
        # Create a book
        test_author = Author.objects.create(first_name='John', last_name='Smith')
        test_genre = Genre.objects.create(name='Fantasy')
        test_language = Language.objects.create(name='English')
        test_book = Book.objects.create(
            title='Book Title',
            summary='My book summary',
            isbn='ABCDEFG',
            author=test_author,
            language=test_language,
        )

        # Create genre as a post-step
        genre_objects_for_book = Genre.objects.all() 
        # Direct assignment of many-to-many types not allowed.
        test_book.genre.set(genre_objects_for_book) 
        test_book.save()

        # Create 30 BookInstance objects
        number_of_book_copies = 30
        for book_copy in range(number_of_book_copies):
            return_date = timezone.now() + datetime.timedelta(days=book_copy%5)
            the_borrower = test_user1 if book_copy % 2 else test_user2
            status = 'm'
            BookInstance.objects.create(book=test_book, imprint='Unlikely Imprint, 2016',
                due_back=return_date, borrower=the_borrower, status=status,
            )
```

---

```pyhon        
    def test_redirect_if_not_logged_in(self):
        response = self.client.get(reverse('my-borrowed'))
        self.assertRedirects(response, '/accounts/login/?next=/catalog/mybooks/')

    def test_logged_in_uses_correct_template(self):
        login = self.client.login(username='testuser1', password='1X<ISRUkw+tuK')
        response = self.client.get(reverse('my-borrowed'))
        
        # Check our user is logged in
        self.assertEqual(str(response.context['user']), 'testuser1')
        # Check that we got a response "success"
        self.assertEqual(response.status_code, 200)

        # Check we used correct template
        self.assertTemplateUsed(response, 
            'catalog/bookinstance_list_borrowed_user.html')
```

</font>

To verify that the view will redirect to a login page if the user is not logged in we use <code>assertRedirects</code>, as demonstrated in <code>test_redirect_if_not_logged_in()</code>. 

---

To verify that the page is displayed for a logged in user we first log in our test user, and then access the page again and check that we get a <code>status_code</code> of 200 (success). 

The rest of the tests verify that our view only returns books that are on loan to our current borrower. Copy the code and paste it onto the end of the test class above.

<font size="5">
  
```python
    def test_only_borrowed_books_in_list(self):
        login = self.client.login(username='testuser1', password='1X<ISRUkw+tuK')
        response = self.client.get(reverse('my-borrowed'))
        
        # Check our user is logged in
        self.assertEqual(str(response.context['user']), 'testuser1')
        # Check that we got a response "success"
        self.assertEqual(response.status_code, 200)
```

---

```python
        # Check that initially we don't have any books in list (none on loan)
        self.assertTrue('bookinstance_list' in response.context)
        self.assertEqual(len(response.context['bookinstance_list']), 0)
        
        # Now change all books to be on loan
        books = BookInstance.objects.all()[:10]

        for book in books:
            book.status = 'o'
            book.save()
        
        # Check that now we have borrowed books in the list
        response = self.client.get(reverse('my-borrowed'))
        # Check our user is logged in
        self.assertEqual(str(response.context['user']), 'testuser1')
        # Check that we got a response "success"
        self.assertEqual(response.status_code, 200)
        
        self.assertTrue('bookinstance_list' in response.context)
        
        # Confirm all books belong to testuser1 and are on loan
        for bookitem in response.context['bookinstance_list']:
            self.assertEqual(response.context['user'], bookitem.borrower)
            self.assertEqual('o', bookitem.status)
```

---

```python
    def test_pages_ordered_by_due_date(self):
        # Change all books to be on loan
        for book in BookInstance.objects.all():
            book.status='o'
            book.save()
            
        login = self.client.login(username='testuser1', password='1X<ISRUkw+tuK')
        response = self.client.get(reverse('my-borrowed'))
        
        # Check our user is logged in
        self.assertEqual(str(response.context['user']), 'testuser1')
        # Check that we got a response "success"
        self.assertEqual(response.status_code, 200)
                
        # Confirm that of the items, only 10 are displayed due to pagination.
        self.assertEqual(len(response.context['bookinstance_list']), 10)
        
        last_date = 0
        for book in response.context['bookinstance_list']:
            if last_date == 0:
                last_date = book.due_back
            else:
                self.assertTrue(last_date <= book.due_back)
                last_date = book.due_back
```

</font>

You could also add pagination tests, should you so wish!

---

##### Testing views with forms

Testing views with forms is a little more complicated than in the cases above, because we need to test more code paths: initial display, display after data validation has failed, and display after validation has succeeded. 

The good news is that we use the client for testing in almost exactly the same way as we did for display-only views.

To demonstrate, let's write some tests for the view used to renew books (<code>renew_book_librarian()</code>):

---

<font size="5">
  
```python
from catalog.forms import RenewBookForm

@permission_required('catalog.can_mark_returned')
def renew_book_librarian(request, pk):
    """View function for renewing a specific BookInstance by librarian."""
    book_instance = get_object_or_404(BookInstance, pk=pk)

    # If this is a POST request then process the Form data
    if request.method == 'POST':

        # Create a form instance and populate it with data from the request 
        # (binding):
        book_renewal_form = RenewBookForm(request.POST)

        # Check if the form is valid:
        if form.is_valid():
            # process the data in form.cleaned_data as required 
            # (here we just write it to the model due_back field)
            book_instance.due_back = form.cleaned_data['renewal_date']
            book_instance.save()

            # redirect to a new URL:
            return HttpResponseRedirect(reverse('all-borrowed'))
```

---

```pyhon
    # If this is a GET (or any other method) create the default form
    else:
        proposed_renewal_date = datetime.date.today() + 
            datetime.timedelta(weeks=3)
        book_renewal_form = RenewBookForm(initial=
            {'renewal_date': proposed_renewal_date})

    context = {
        'book_renewal_form': book_renewal_form,
        'book_instance': book_instance,
    }

    return render(request, 'catalog/book_renew_librarian.html', context)
```

</font>

We'll need to test that the view is only available to users who have the <code>can_mark_returned</code> permission, and that users are redirected to an HTTP 404 error page if they attempt to renew a <code>BookInstance</code> that does not exist. 

---

We should check that the initial value of the form is seeded with a date three weeks in the future, and that if validation succeeds we're redirected to the "all-borrowed books" view. 

As part of checking the validation-fail tests we'll also check that our form is sending the appropriate error messages.

Add the first part of the test class to the bottom of <b>/catalog/tests/test_views.py</b>.

This creates two users and two book instances, but only gives one user the permission required to access the view. The code to grant permissions during tests is shown:

---

<font size="5">
  
```python
import uuid

# Required to grant the permission needed to set a book as returned.
from django.contrib.auth.models import Permission 

class RenewBookInstancesViewTest(TestCase):
    def setUp(self):
        # Create a user
        test_user1 = User.objects.create_user(username='testuser1', 
            password='1X<ISRUkw+tuK')
        test_user2 = User.objects.create_user(username='testuser2', 
            password='2HJ1vRV0Z&3iD')

        test_user1.save()
        test_user2.save()
        
        permission = Permission.objects.get(name='Set book as returned')
        test_user2.user_permissions.add(permission)
        test_user2.save()

        # Create a book
        test_author = Author.objects.create(first_name='John', last_name='Smith')
        test_genre = Genre.objects.create(name='Fantasy')
        test_language = Language.objects.create(name='English')
        test_book = Book.objects.create(
            title='Book Title',
            summary='My book summary',
            isbn='ABCDEFG',
            author=test_author,
            language=test_language,
        )
```

---

```python        
        # Create genre as a post-step
        genre_objects_for_book = Genre.objects.all() 
        # Direct assignment of many-to-many types not allowed.
        test_book.genre.set(genre_objects_for_book) 
        test_book.save()

        # Create a BookInstance object for test_user1
        return_date = datetime.date.today() + datetime.timedelta(days=5)
        self.test_bookinstance1 = BookInstance.objects.create(
            book=test_book,
            imprint='Unlikely Imprint, 2016',
            due_back=return_date,
            borrower=test_user1,
            status='o',
        )

        # Create a BookInstance object for test_user2
        return_date = datetime.date.today() + datetime.timedelta(days=5)
        self.test_bookinstance2 = BookInstance.objects.create(
            book=test_book,
            imprint='Unlikely Imprint, 2016',
            due_back=return_date,
            borrower=test_user2,
            status='o',
        )
```

</font>

---

Add the tests to the bottom of the test class. 

These check that only users with the correct permissions (testuser2) can access the view. 

We check all the cases: 

<font size="6">

- when the user is not logged in, 
- when a user is logged in but does not have the correct permissions, 
- when the user has permissions but is not the borrower (should succeed), and 
- what happens when they try to access a <code>BookInstance</code> that doesn't exist. 
- the correct template is used.

</font>

---

<font size="5">
  
```python
    def test_redirect_if_not_logged_in(self):
    
        response = self.client.get(reverse('renew-book-librarian', 
            kwargs={'pk': self.test_bookinstance1.pk}))            
        # Manually check redirect (Can't use assertRedirect, 
        # because the redirect URL is unpredictable)
        self.assertEqual(response.status_code, 302)
        self.assertTrue(response.url.startswith('/accounts/login/'))
        
    def test_redirect_if_logged_in_but_not_correct_permission(self):
        login = self.client.login(username='testuser1', password='1X<ISRUkw+tuK')
        response = self.client.get(reverse('renew-book-librarian', 
            kwargs={'pk': self.test_bookinstance1.pk}))
        self.assertEqual(response.status_code, 403)

    def test_logged_in_with_permission_borrowed_book(self):
        login = self.client.login(username='testuser2', password='2HJ1vRV0Z&3iD')
        response = self.client.get(reverse('renew-book-librarian', 
            kwargs={'pk': self.test_bookinstance2.pk}))
        
        # Check that it lets us login - this is our book and 
        # we have the right permissions.
        self.assertEqual(response.status_code, 200)
```

---

```python
    def test_logged_in_with_permission_another_users_borrowed_book(self):
        login = self.client.login(username='testuser2', password='2HJ1vRV0Z&3iD')
        response = self.client.get(reverse('renew-book-librarian', 
            kwargs={'pk': self.test_bookinstance1.pk}))

        # Check that it lets us login. We're a librarian, so we can view any users book
        self.assertEqual(response.status_code, 200)

    def test_HTTP404_for_invalid_book_if_logged_in(self):
        # unlikely UID to match our bookinstance!
        test_uid = uuid.uuid4()
        login = self.client.login(username='testuser2', password='2HJ1vRV0Z&3iD')
        response = self.client.get(reverse('renew-book-librarian', 
            kwargs={'pk':test_uid}))
        self.assertEqual(response.status_code, 404)
```

---

```python        
    def test_uses_correct_template(self):
        login = self.client.login(username='testuser2', password='2HJ1vRV0Z&3iD')
        response = self.client.get(reverse('renew-book-librarian', 
            kwargs={'pk': self.test_bookinstance1.pk}))
        self.assertEqual(response.status_code, 200)

        # Check we used correct template
        self.assertTemplateUsed(response, 'catalog/book_renew_librarian.html')

        test_user1.save()
        test_user2.save()
        
        permission = Permission.objects.get(name='Set book as returned')
        test_user2.user_permissions.add(permission)
        test_user2.save()

        # Create a book
        test_author = Author.objects.create(first_name='John', last_name='Smith')
        test_genre = Genre.objects.create(name='Fantasy')
        test_language = Language.objects.create(name='English')
        test_book = Book.objects.create(
            title='Book Title',
            summary='My book summary',
            isbn='ABCDEFG',
            author=test_author,
            language=test_language,
        )
```

</font>

---

Add the next test method. This checks that the initial date for the form is three weeks in the future. Note how we are able to access the value of the initial value of the form field.

<font size="5">
  
```python        
    def test_form_renewal_date_initially_has_date_three_weeks_in_future(self):
        login = self.client.login(username='testuser2', password='2HJ1vRV0Z&3iD')
        response = self.client.get(reverse('renew-book-librarian', 
            kwargs={'pk': self.test_bookinstance1.pk}))
        self.assertEqual(response.status_code, 200)
        
        date_3_weeks_in_future = datetime.date.today() + 
            datetime.timedelta(weeks=3)
        self.assertEqual(response.context['form'].
            initial['renewal_date'], date_3_weeks_in_future)
```

</font>

---

The next test (add this to the class too) checks that the view redirects to a list of all borrowed books if renewal succeeds. 

What differs here is that for the first time we show how you can <code>POST</code> data using the client. 

The post data is the second argument to the post function, and is specified as a dictionary of key/values.

<font size="5">
  
```python        
    def test_redirects_to_all_borrowed_book_list_on_success(self):
        login = self.client.login(username='testuser2', password='2HJ1vRV0Z&3iD')
        valid_date_in_future = datetime.date.today() + datetime.timedelta(weeks=2)
        response = self.client.post(reverse('renew-book-librarian', 
            kwargs={'pk':self.test_bookinstance1.pk,}
            {'renewal_date':valid_date_in_future})
        self.assertRedirects(response, reverse('all-borrowed'))
```

</font>

---

Copy the last two functions into the class. These again test <code>POST</code> requests, but in this case with invalid renewal dates. We use <code>assertFormError()</code> to verify that the error messages are as expected.

<font size="5">
  
```python        
    def test_form_invalid_renewal_date_past(self):
        login = self.client.login(username='testuser2', password='2HJ1vRV0Z&3iD')
        date_in_past = datetime.date.today() - datetime.timedelta(weeks=1)
        response = self.client.post(reverse('renew-book-librarian', 
            kwargs={'pk': self.test_bookinstance1.pk}), 
                   {'renewal_date': date_in_past})
        self.assertEqual(response.status_code, 200)
        self.assertFormError(response, 'form', 'renewal_date', 
            'Invalid date - renewal in past')
        
    def test_form_invalid_renewal_date_future(self):
        login = self.client.login(username='testuser2', password='2HJ1vRV0Z&3iD')
        invalid_date_in_future = datetime.date.today() + 
            datetime.timedelta(weeks=5)
        response = self.client.post(reverse('renew-book-librarian', 
            kwargs={'pk': self.test_bookinstance1.pk}), 
                   {'renewal_date': invalid_date_in_future})
        self.assertEqual(response.status_code, 200)
        self.assertFormError(response, 'form', 'renewal_date', 
        'Invalid date - renewal more than 4 weeks ahead')
```

</font>

---

#### Templates

Django provides test APIs to check that the correct template is being called by our views, and to allow us to verify that the correct information is being sent. 

There is however no specific API support for testing in Django that our HTML output is rendered as expected.

---

### Other recommended test tools

Django's test framework can help us write effective unit and integration tests — we've only scratched the surface of what the underlying <b>unittest</b> framework can do, let alone Django's additions (for example, check out how we can use <u>unittest.mock</u> to patch third party libraries so we can more thoroughly test our own code).

---

While there are numerous other test tools that we can use, we'll just highlight two:

<font size="6">
  
- <u>Coverage</u>: This Python tool reports on how much of our code is actually executed by our tests. It is particularly useful when we're getting started, and we are trying to work out exactly what we should test.
- <u>Selenium</u> is a framework to automate testing in a real browser. It allows us to simulate a real user interacting with the site, and provides a great framework for system testing our site (the next step up from integration testing).
</font>

---

### Challenge yourself

There are a lot more models and views we can test. As a simple task, try to create a test case for the AuthorCreate view.

<font size="6">
  
```python        
class AuthorCreate(PermissionRequiredMixin, CreateView):
    model = Author
    fields = '__all__'
    initial = {'date_of_death':'12/10/2016'}
    permission_required = 'catalog.can_mark_returned'
```

</font>

Remember that you need to check anything that you specify or that is part of the design. This will include who has access, the initial date, the template used, and where the view redirects on success.




