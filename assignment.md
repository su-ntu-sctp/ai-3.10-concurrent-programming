# Assignment (Optional)

## Brief

Create a program called MultithreadingAssignment.java and solve the following problems using threads, synchronization, ExecutorService, and CompletableFuture.

1. **Bank Account with Multiple Threads and Synchronization**
   - Create a `BankAccount` class with:
     - `private double balance` attribute
     - Constructor that initializes balance to 1000
     - `synchronized void deposit(double amount)` method
     - `synchronized void withdraw(double amount)` method
     - Both methods should print the current balance after each transaction
   - Create two `Runnable` tasks:
     - `depositTask`: Deposits 100 five times (total 500)
     - `withdrawTask`: Withdraws 50 five times (total 250)
   - Use `ExecutorService` with a fixed thread pool of 4 threads
   - Submit the deposit task twice and the withdraw task twice to the executor
   - Wait for all tasks to complete and print the final balance
   - Expected final balance: 1000 + (2 × 500) - (2 × 250) = 1500
   - Demonstrate that without `synchronized`, the balance may be incorrect (comment out synchronized and show the issue)

2. **Asynchronous Task Processing with CompletableFuture**
   - Create three methods that simulate time-consuming operations:
     - `fetchUserData()`: Returns a String "User: John Doe" after a 2-second delay
     - `fetchOrderData()`: Returns a String "Orders: 5" after a 3-second delay
     - `calculateTotal()`: Returns an Integer 250 after a 1-second delay
   - Use `Thread.sleep()` to simulate the delays
   - Use `CompletableFuture.supplyAsync()` to run all three methods asynchronously
   - Use `thenAccept()` to print each result when it completes
   - Use `CompletableFuture.allOf()` to wait for all tasks to complete
   - Add exception handling using `exceptionally()` for at least one task
   - Measure and print the total execution time (should be approximately 3 seconds, not 6 seconds)
   - **Bonus**: Try the same implementation using virtual threads (Java 21) and compare

## Submission (Optional)

- Submit the URL of the GitHub Repository that contains your work to NTU black board.
- Should you reference the work of your classmate(s) or online resources, give them credit by adding either the name of your classmate or URL.

## References
- Java: https://docs.oracle.com/javase/
- Spring Boot: https://docs.spring.io/spring-boot/docs/current/reference/html/
- PostgreSQL: https://www.postgresql.org/docs/
- OWASP: https://cheatsheetseries.owasp.org/