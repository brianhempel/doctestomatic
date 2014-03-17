# Doctestomatic

Doctestomatic facilitates Documentation Driven Development by allowing you to define expectations in your documentation which can be pulled into your test suite.

Doctestomatic is currently an experiment to see if this is even a good idea.

The expectations in the documentation do not need to be comprehensive. You can add more tests in your tests.

## What you could gain by using Doctestomatic

1. Confidence that your documentation is telling the truth.
2. A process that writes documenation first, so it's not a drag at the end.
3. Which that, it's less of a drag to document edge cases. 

Basically, Doctestomatic is a ploy for developer happiness and better software.

## What you could lose by using Doctestomatic

1. Some expectations will not be grouped with your tests, so you may be switching files.
2. It could be clumsy.

## How it works

You write your docs in HTML or Markdown or whatever. Whenever you come to something that could be the subject or an expection for a test, you drop into ERB and write a bit of code.

When generating your docs, all the ERB code resolves to the strings you want to see in the documenation.

In the tests, the ERB files are evaluated and the tests pluck out the subjects and expectations and use them to verify your application.

Here's an example.

```html+erb
<h1>Birdsong API</h1>
<p>
  To list all the birdsongs:
</p>
<pre>
  curl -i https://birdsongapi.org<%= subject(:index) { "/songs" } %>
</pre>
<p>
  You should get something back, like:
</p>
<pre class="headers">
  <%= subject.let(:expected_headers) { dt_headers({"Content-type" => "application/json" }) } %>
</pre>
<pre class="json">
<%= subject.let(:expected_json) { dt_json(
  {
    songs: dt_json_two_or_more_element_array(
      [
        {
          id:   dt_natural_number(1),
          bird: dt_present_string("Black-capped Chickadee")
        },
        {
          id:   dt_natural_number(2),
          bird: dt_present_string("White-breasted Nuthatch")
        },
        {
          id:   dt_natural_number(3),
          bird: dt_present_string("Gray Jay")
        }
      ]
    )
  }
) } %>
</pre>
```

When you render the ERB to make your documentation, the matchers produce the expected example output.

```ruby
require 'doctestomatic'

songs_file = File.read("doc/songs.html.erb")
puts Doctestomatic::Doctest.new(songs_file).result
```
```html
<h1>Birdsong API</h1>
<p>
  To list all the birdsongs:
</p>
<pre>
  curl -i https://birdsongapi.org/songs
</pre>
<p>
  You should get something back, like:
</p>
<pre class="headers">
  Content-type: application/json
</pre>
<pre class="json">
{
  "songs": [
    {
      id: 1,
      bird: "Black-capped Chickadee"
    },
    {
      id: 2,
      bird: "White-breasted Nuthatch"
    },
    {
      id: 3,
      bird: "Gray Jay"
    }
  ]
}
</pre>
</body>
</html>
```

When you pull the file into your tests, you have access to all the subjects and lets that you set. You are not dependent on any particular testing framework. The following example uses [Rack::Test](https://github.com/brynary/rack-test) in [Minitest](https://github.com/seattlerb/minitest).

```ruby
require 'doctastic'
require 'rack/test'
require 'minitest/autorun'

# produce the kind of errors Minitest expects
Doctestomatic.error_factory do |message|
  raise(Minitest::Assertion, message)
end

MY_APP = Rack::Builder.parse_file('config.ru').first

class TestBirdsongAPI < Minitest::Test
  include Rack::Test::Methods

  def app
    MY_APP
  end

  def self.doctest
    Doctestomatic::Doctest.new("docs/songs.html.erb")
  end

  doctest.subjects_and_lets.each do |subject, lets|
    define_method "test_#{subject.name}_headers" do
      get(subject.to_s) # "/songs"

      expected_headers = lets(:expected_headers)
      actual_headers   = last_response.headers

      expected_headers.assert_match(actual_headers)
    end

    define_method "test_#{subject.name}_json" do
      get(subject.to_s) # "/songs"

      expected_json = lets(:expected_json)
      actual_json   = JSON.parse(last_response.body)

      expected_json.assert_match(actual_json)
    end
  end

  #
  # You are not limited to the tests in your docs,
  # you can add further tests here.
  #

end
```

  
