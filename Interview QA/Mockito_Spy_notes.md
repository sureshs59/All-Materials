In Mockito, spy() is used to create a partial mock of a real object. Unlike a mock, which returns default values unless stubbed, a spy calls the actual methods by default. You can selectively stub specific methods while allowing the rest to execute normally.

Mock vs Spy
Mock	Spy
No real object is used	Wraps a real object
All methods return default values unless stubbed	Real methods are called unless stubbed
Used when you want to completely isolate the class	Used when you want to override only certain behavior
Example 1: Basic Spy
import static org.mockito.Mockito.*;

import java.util.ArrayList;
import java.util.List;

public class SpyExample {
    public static void main(String[] args) {

        List<String> realList = new ArrayList<>();

        List<String> spyList = spy(realList);

        spyList.add("Apple");
        spyList.add("Banana");

        System.out.println(spyList.size());      // Calls real method -> 2
        System.out.println(spyList.get(0));      // Apple

        verify(spyList).add("Apple");
        verify(spyList).add("Banana");
    }
}

Output

2
Apple

Since it's a spy, the real ArrayList methods are executed.

Example 2: Stubbing a Method
List<String> list = new ArrayList<>();
List<String> spyList = spy(list);

spyList.add("Apple");
spyList.add("Banana");

// Override only size()
doReturn(100).when(spyList).size();

System.out.println(spyList.size());   // 100
System.out.println(spyList.get(0));   // Apple

Output

100
Apple

Here:

size() is mocked.
get() still calls the real method.
Example 3: Why doReturn() is Preferred with Spy

Suppose:

List<String> spyList = spy(new ArrayList<>());

If you write:

when(spyList.get(0)).thenReturn("Hello");

Mockito first executes:

spyList.get(0);

Since the list is empty, you'll get:

IndexOutOfBoundsException

Instead, use:

doReturn("Hello").when(spyList).get(0);

Now:

System.out.println(spyList.get(0));

Output:

Hello

doReturn() avoids calling the real method during stubbing.

Example 4: Real-World Example
class PaymentService {

    public String processPayment() {
        return validate() + " Payment Successful";
    }

    public String validate() {
        return "Validated";
    }
}

Test:

PaymentService service = spy(new PaymentService());

doReturn("Mock Validation")
        .when(service)
        .validate();

String result = service.processPayment();

System.out.println(result);

Output:

Mock Validation Payment Successful

What happened:

processPayment() is the real implementation.
Inside it, validate() is intercepted by the spy and returns the stubbed value.
When to Use spy()

Use a spy when:

You want to test most of the real implementation.
You need to override only a few methods.
You're working with legacy code that's difficult to fully mock.
You want to verify interactions while still executing real logic.

Avoid using spies when a regular mock is sufficient, as spies can make tests more tightly coupled to implementation details.

Summary
mock() creates a fake object; all methods are mocked by default.
spy() wraps a real object; real methods run unless explicitly stubbed.
Prefer doReturn(...).when(spy)... instead of when(...).thenReturn(...) when stubbing spies to avoid invoking the real method during setup.
Spies are useful for partial mocking, where you want a combination of real behavior and mocked behavior.