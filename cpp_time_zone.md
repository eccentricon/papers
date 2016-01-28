# C++ Standard Proposal &mdash; A Time Zone Library

Authors: Greg Miller (jgm@google.com), Bradley White (bww@google.com)

## Motivation

Programming with time on a human scale is notoriously difficult and error prone:
time zones are complicated, daylight-saving time (DST) is complicated, calendars
are complicated, and leap seconds are complicated. These complexities quickly
surface in code because programmers do not have a simple conceptual model with
which to reason about the time-programming challenges they are facing. This lack
of a simple conceptual model begets the lack of a simple time-programming
library, leaving only complicated libraries that programmers struggle to
understand and use correctly.

A few years ago we set out to fix these problems within Google by doing the
following:

* Defining a simple conceptual model that will help programmers reason about
  arbitrarily complex situations involving time, time zones, DST, etc.
* Producing a simple library (or two) that implements the conceptual model.

This paper describes the Time Zone Library that has been widely used within
Google for a couple years. Our goal with this paper is to inform the C++
Standards Committee about the design and trade-offs we considered and the
results of our real-world usage.

NOTE: This paper depends on the related paper proposing a standard Civil Time
Library (XXX: jgm add a link here).

## Conceptual Model

The conceptual model for time-programming that we teach within Google consists
of three simple concepts that we will define here (Note: this model and these
definitions are not specific to C++).

*Absolute time* uniquely and universally represents a specific instant in time.
It has no notion of calendars, or dates, or times of day. Instead, it is a
measure of the passage of real time, typically as a simple count of ticks since
some epoch. Absolute times are independent of all time zones and do not suffer
from human-imposed complexities such as daylight-saving time (DST). Many C++
types exist to represent absolute times, classically `time_t` and more recently
`std::chrono::time_point`.

*Civil time* is the legally recognized representation of time for ordinary
affairs (cf. http://www.merriam-webster.com/dictionary/civil). It is a
human-scale representation of time that consists of the six fields &mdash;
year, month, day, hour, minute, and second (sometimes shortened to "YMDHMS")
&mdash; and it follows the rules of the [Proleptic Gregorian Calendar], with
24-hour days divided into hours and minutes. Like absolute times, civil times
are also independent of all time zones and their related complexities (e.g.,
DST). While `std::tm` contains the six YMDHMS civil-time fields (plus a few
more), it does not have behavior to enforce the rules of civil time as just
described.

*Time zones* are geo-political regions within which human-defined rules are
shared to convert between the previously described absolute time and civil time
domains. A time-zone's rules include things like the region's offset from the
[UTC] time standard, daylight-saving adjustments, and short abbreviation
strings. Time zones often have a history of disparate rules that apply only for
certain periods because the rules may change at the whim of a region's local
government. For this reason, time zone rules are often compiled into data
snapshots that are used at runtime to perform conversions between absolute and
civil times. A proposal for a standard time zone library is presented in
another paper (XXX: jgm add a link here).

The C++ standard library already has `<chrono>`, which is a good implementation
of *absolute time* (as well as the related duration concept). Another paper is
proposing a standard Civil Time Library that complements `<chrono>` (XXX: jgm
insert link to civil time paper). This paper is proposing a standard *time
zone* library that follows the complex rules defined by time zones and provides
a mapping between the absolute and civil time domains.

## Overview

Time zones are canonically identified by a string of the form
[Continent]/[City], such as "America/New_York", "Europe/London", and
"Australia/Sydney". The data encapsulated by a time zone describes the offset
from the [UTC] time standard (in seconds east), a short abbreviation string
(e.g., "EST", "PDT"), and information about daylight-saving time (DST). These
rules are defined by local governments and they may change over time. A time
zone, therefore, represents the complete history of time zone rules and when
each rule applies for a given region.

Ultimately, a time zone represents the rules necessary to convert any *absolute
time* to a *civil time* and vice versa.

In this model, [UTC] itself is naturally represented as a time zone having a
constant zero offset, no DST, and an abbreviation string of "UTC". Treating UTC
like any other time zone enables programmers to write correct,
time-zone-agnostic code without needing to special-case UTC.

The core of the Time Zone Library presented here is a single class named
`time_zone`, which has two member functions to convert between absolute time and
civil time. Absolute times are represented by `std::chrono::time_point` (on the
system_clock), and civil times are represented using `civil_second` as described
in the proposed Civil Time Library (XXX: jgm add link to that paper). The Time
Zone Library also defines a convenience syntax for doing conversions through a
time zone. There are also functions to format and parse absolute times as
strings.

## API

The interface for the core `time_zone` class is as follows.

```cpp
#include <chrono>
#include "civil.h"  // XXX: jgm reference the other paper

// Convenience aliases.
template <typename D>
using time_point = std::chrono::time_point<std::chrono::system_clock, D>;
using sys_seconds = std::chrono::duration<std::chrono::system_clock::rep,
                                          std::chrono::seconds::period>;

class time_zone {
 public:
  // A value type.
  time_zone() = default;  // Equivalent to UTC
  time_zone(const time_zone&) = default;
  time_zone& operator=(const time_zone&) = default;

  struct time_conversion {
    civil_second cs;
    int offset;        // seconds east of UTC
    bool is_dst;       // is offset non-standard?
    std::string abbr;  // time-zone abbreviation (e.g., "PST")
  };
  template <typename D>
  time_conversion convert(const time_point<D>& tp) const;

  struct civil_conversion {
    enum class kind {
      UNIQUE,    // the civil time was singular (pre == trans == post)
      SKIPPED,   // the civil time did not exist
      REPEATED,  // the civil time was ambiguous
    } kind;
    time_point<sys_seconds> pre;   // Uses the pre-transition offset
    time_point<sys_seconds> trans;
    time_point<sys_seconds> post;  // Uses the post-transition offset
  };
  civil_conversion convert(const civil_second& cs) const;

 private:
  ...
};

// Loads the named time zone. Returns false on error.
bool load_time_zone(const std::string& name, time_zone* tz);

// Returns a time_zone representing UTC.
time_zone utc_time_zone();

// Returns a time zone representing the local time zone.
time_zone local_time_zone();
```

Converting from an absolute time to a civil time (e.g.,
`std::chrono::time_point` to `civil_second`) is an exact calculation with no
possible time zone ambiguities. However, conversion from civil time to absolute
time may not be exact. Conversions around UTC offset transitions may be given
ambiguous civil times (e.g., the 1:00 am hour is repeated during the Autumn DST
transition in the United States), and some civil times may not exist in a
particular time zone (e.g., the 2:00 am hour is skipped during the Spring DST
transition in the United States). The `time_zone::civil_conversion` struct gives
callers all relevant information about the conversion operation.

However, the full information provided by the `time_zone::time_conversion` and
`time_zone::civil_conversion` structs is frequently not needed by callers. To
simplify the common case of converting between `std::chrono::time_point` and
`civil_second`, the Time Zone Library provides two overloads of `operator|` to
allow "piping" either type to a `time_zone` in order to convert to the other
type.

The implementation of these convenience functions must select an appropriate
"default" time point to return in cases of ambiguous/skipped civil time
conversions. The value chosen is such that the relative ordering of civil times
is preserved when they are converted to absolute times.

Note: This convenience syntax exists to shorten common code samples, and to
select a generally good default for programmers when necessary. It is not an
essential part of the Time Zone Library proposed in this paper.

```cpp
template <typename D>
inline CivilSecond operator|(const time_point<D>& tp, const time_zone& tz) {
  return tz.convert(tp).cs;
}

inline time_point<sys_seconds> operator|(const civil_second& cs, const time_zone& tz) {
  const auto conv = tz.convert(cs);
  if (conv.kind == time_zone::civil_conversion::kind::SKIPPED)
    return conv.trans;
  return conv.pre;
}
```

Finally, functions are provided for formatting and parsing absolute times with
respect to a given time zone. These functions use `strftime()`-like format
specifiers, with the following extensions:

Specifier | Description
----------|------------
`%Ez`     | RFC3339-compatible numeric time zone (+hh:mm or -hh:mm)
`%E#S`    | Seconds with # digits of fractional precision
`%E*S`    | Seconds with full fractional precision (a literal '*')
`%E4Y`    | Four-character years (-999 ... -001, 0000, 0001 ... 9999)

```cpp
template <typename D>
std::string format(const std::string& format, const time_point<D>& tp,
                   const time_zone& tz);

template <typename D>
bool parse(const std::string& format, const std::string& input,
           const time_zone& tz, time_point<D>* tpp);
```

## Examples


[Proleptic Gregorian Calendar]: https://en.wikipedia.org/wiki/Proleptic_Gregorian_calendar
[UTC]: https://en.wikipedia.org/wiki/Coordinated_Universal_Time