# 600 - Practice refactoring legacy code with tests

# Deep Dive: Refactoring Legacy Code with Tests

## The Legacy Code Challenge

Legacy code in enterprise environments presents a unique challenge: you need to modify or fix code that has no tests, but adding tests is difficult because the code wasn’t designed to be testable. This creates a catch-22 situation that requires specific strategies to break the cycle.

## The Golden Rule: Make It Testable First

The fundamental principle when working with legacy code is **you cannot refactor safely without tests, but you need to refactor to make code testable**. The solution is to make minimal, surgical changes to enable testing, then add tests, then refactor properly.

## Step-by-Step Workflow

### Phase 1: Identify the Change Point

Start by identifying exactly where you need to make changes. Don’t try to test everything at once. Focus on the specific area you need to modify. This is called finding your “change point” - the seam where you’ll insert new behavior or fix a bug.

### Phase 2: Find the Seams

A “seam” is a place where you can alter behavior without editing the code in that location. In Python, common seams include:

**Dependency Injection Points** - Where you can pass in collaborators rather than hard-coding them:

```python
# Hard to test (legacy)
class OrderProcessor:
    def process(self, order):
        db = DatabaseConnection('prod_server')  # Hard-coded dependency
        result = db.save(order)
        EmailService().send_confirmation(order.email)  # Hard-coded
        
# Better (testable)
class OrderProcessor:
    def __init__(self, db=None, email_service=None):
        self.db = db or DatabaseConnection('prod_server')
        self.email_service = email_service or EmailService()
    
    def process(self, order):
        result = self.db.save(order)
        self.email_service.send_confirmation(order.email)
```

**Monkey Patching Seams** - Using pytest’s monkeypatch or unittest.mock to intercept calls:

```python
# Legacy function you can't easily change
def process_payment(amount):
    api = PaymentGateway()  # External API call
    return api.charge(amount)

# Test using monkeypatch
def test_process_payment(monkeypatch):
    mock_gateway = Mock()
    mock_gateway.charge.return_value = {'status': 'success'}
    monkeypatch.setattr('payment_module.PaymentGateway', 
                        lambda: mock_gateway)
    result = process_payment(100)
    assert result['status'] == 'success'
```

### Phase 3: Write Characterization Tests

Characterization tests (also called “approval tests” or “golden master tests”) capture the *current* behavior of the code, even if that behavior is wrong. The goal is to lock down existing behavior so you can refactor safely.

**The Process:**

1. Write a test with a guess at what the output should be
1. Run the test and let it fail
1. Examine the actual output
1. If the output represents actual current behavior, update your test to expect that output
1. Now you have a safety net

**Example:**

```python
# Legacy function with unclear behavior
def calculate_discount(price, customer_type, is_holiday):
    if customer_type == "premium":
        discount = price * 0.2
        if is_holiday:
            discount = discount * 1.5
    elif customer_type == "regular":
        discount = price * 0.1
    else:
        discount = 0
    return price - discount

# Characterization tests
def test_premium_customer_regular_day():
    result = calculate_discount(100, "premium", False)
    assert result == 80  # Captures current behavior

def test_premium_customer_holiday():
    result = calculate_discount(100, "premium", True)
    assert result == 70  # Wait, is this right? Test will tell us!

def test_regular_customer():
    result = calculate_discount(100, "regular", False)
    assert result == 90

def test_unknown_customer_type():
    result = calculate_discount(100, "unknown", False)
    assert result == 100  # No discount applied
```

### Phase 4: The Sprout Technique

When you need to add new functionality to legacy code, don’t weave it into the untested mess. Instead, write the new functionality in a new, well-tested function, then call it from the legacy code.

**Example:**

```python
# Legacy code
def process_order(order_data):
    # 200 lines of untested spaghetti code
    validate_inventory(order_data)
    calculate_total(order_data)
    charge_customer(order_data)
    # ... more complexity
    
# You need to add fraud detection
# DON'T modify process_order directly!

# Instead, create a new, testable function
def detect_fraud(order_data):
    """New function with tests"""
    risk_score = 0
    if order_data['amount'] > 10000:
        risk_score += 50
    if order_data['shipping_country'] != order_data['billing_country']:
        risk_score += 30
    return risk_score > 70

# Comprehensive tests for the new function
def test_detect_fraud_high_amount():
    order = {'amount': 15000, 'shipping_country': 'US', 
             'billing_country': 'US'}
    assert detect_fraud(order) == False

def test_detect_fraud_country_mismatch_high_amount():
    order = {'amount': 15000, 'shipping_country': 'US', 
             'billing_country': 'CN'}
    assert detect_fraud(order) == True

# Then add a single line to legacy code
def process_order(order_data):
    if detect_fraud(order_data):  # One line addition
        raise FraudException()
    # ... rest of legacy code unchanged
```

### Phase 5: The Wrap Technique

When you can’t easily modify a class, wrap it in a new interface that’s testable:

```python
# Legacy class - untestable
class LegacyReportGenerator:
    def generate(self):
        # Talks directly to database, filesystem, etc.
        data = self._query_database()
        self._write_to_file(data)
        self._send_email()
        
# Wrapper that extracts testable logic
class ReportGenerator:
    def __init__(self, legacy_generator=None, data_source=None, 
                 output=None):
        self.legacy = legacy_generator or LegacyReportGenerator()
        self.data_source = data_source
        self.output = output
    
    def generate(self):
        # New, testable logic
        if self.data_source:
            data = self.data_source.get_data()
        else:
            data = self.legacy._query_database()
        
        formatted = self._format_data(data)  # New, pure function
        
        if self.output:
            self.output.write(formatted)
        else:
            self.legacy._write_to_file(formatted)
    
    def _format_data(self, data):
        # Pure function - easy to test!
        return [{'name': d.name.upper(), 'value': d.value} 
                for d in data]

# Now you can test the formatting logic
def test_format_data():
    generator = ReportGenerator()
    data = [Mock(name='john', value=100)]
    result = generator._format_data(data)
    assert result == [{'name': 'JOHN', 'value': 100}]
```

## Advanced Strategies

### Breaking Dependencies

**Extract and Override:** Create a subclass for testing that overrides problematic methods:

```python
class LegacyService:
    def process(self):
        result = self._call_external_api()  # Can't test this
        return self._transform(result)
    
    def _call_external_api(self):
        return requests.get('https://api.example.com/data')
    
    def _transform(self, data):
        # Logic we want to test
        return data.upper()

# Test subclass
class TestableService(LegacyService):
    def _call_external_api(self):
        return "test data"  # Override with test data

def test_transform_logic():
    service = TestableService()
    result = service.process()
    assert result == "TEST DATA"
```

### Introduce Parameter

Transform hard-coded dependencies into parameters:

```python
# Before - untestable
def send_notification(user_id):
    user = User.objects.get(id=user_id)  # Database call
    if user.email:
        EmailService().send(user.email, "Hello")

# After - testable (gradual improvement)
def send_notification(user_id, user_repo=None):
    repo = user_repo or UserRepository()  # Default to real implementation
    user = repo.get(user_id)
    if user.email:
        EmailService().send(user.email, "Hello")

# Test
def test_send_notification():
    mock_repo = Mock()
    mock_repo.get.return_value = Mock(email='test@example.com')
    send_notification(123, user_repo=mock_repo)
    # Can verify behavior without database
```

## Practical Refactoring Patterns

### The Strangler Fig Pattern

Gradually replace legacy code by building new code alongside it, routing traffic to the new code incrementally:

```python
# Phase 1: Legacy code still in use
def calculate_price(item):
    # Old, complex logic
    pass

# Phase 2: New code introduced
def calculate_price_v2(item):
    # New, tested logic
    pass

def calculate_price(item):
    if feature_flag('use_new_pricing'):
        return calculate_price_v2(item)
    return old_calculate_price(item)

# Phase 3: Eventually remove old code entirely
```

### Extract Method

Break large methods into smaller, testable pieces:

```python
# Before: 100-line method
def process_customer_order(order):
    # Validate customer
    # Check inventory
    # Calculate shipping
    # Apply discounts
    # Process payment
    # Send confirmation
    # Update analytics
    # ... 100 lines

# After: Extracted methods
def process_customer_order(order):
    customer = validate_customer(order.customer_id)
    inventory = check_inventory(order.items)
    shipping = calculate_shipping(order.address, inventory)
    total = apply_discounts(order, customer)
    payment = process_payment(customer, total)
    send_confirmation(customer, order, payment)
    update_analytics(order)

# Each extracted method can now be tested independently
def test_calculate_shipping():
    address = {'country': 'US', 'zip': '12345'}
    inventory = [{'weight': 5, 'dimensions': '10x10x10'}]
    cost = calculate_shipping(address, inventory)
    assert cost == 15.99
```

## Tools for Legacy Code Testing

**Coverage Analysis:**

- **coverage.py** with pytest-cov - Identify untested code paths
- Run with `pytest --cov=myapp --cov-report=html` to see visual coverage reports
- Focus on adding tests where coverage is low AND risk is high

**Mutation Testing:**

- **mutmut** - Introduces bugs to verify your tests catch them
- Helps identify weak tests that pass even when code is broken
- Run: `mutmut run` to see if your tests are effective

**Approval Testing:**

- **approvaltests** - Compare output against approved baseline
- Great for complex outputs (JSON, HTML, reports)

```python
from approvaltests import verify

def test_report_generation():
    report = generate_complex_report(sample_data)
    verify(report)  # First run creates baseline, subsequent runs compare
```

**Static Analysis:**

- **pylint**, **flake8** - Find code smells before testing
- **mypy** - Add type hints gradually to improve testability
- **radon** - Measure cyclomatic complexity (high complexity = hard to test)

## Exercise: Complete Refactoring Example

Here’s a realistic legacy code scenario and how to systematically improve it:

```python
# LEGACY CODE - Don't judge, we've all seen worse!
class OrderProcessor:
    def process_order(self, order_id):
        # Get order from database
        conn = sqlite3.connect('orders.db')
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM orders WHERE id=?", (order_id,))
        order = cursor.fetchone()
        
        # Calculate total
        total = 0
        for item_id in order[3].split(','):
            cursor.execute("SELECT price FROM items WHERE id=?", (item_id,))
            price = cursor.fetchone()[0]
            total += price
        
        # Apply discount
        if order[2] == 'premium':
            total = total * 0.9
        
        # Charge credit card
        card_number = order[4]
        import stripe
        stripe.api_key = 'sk_live_...'
        charge = stripe.Charge.create(amount=total, currency='usd',
                                       source=card_number)
        
        # Send email
        import smtplib
        server = smtplib.SMTP('smtp.gmail.com', 587)
        server.starttls()
        server.login('myemail@gmail.com', 'password123')
        server.sendmail('myemail@gmail.com', order[1],
                        f'Order confirmed! Total: ${total}')
        
        conn.close()
        return charge.id
```

**Step 1: Add Characterization Test (using mocks extensively)**

```python
def test_process_order_characterization(monkeypatch):
    # Mock database
    mock_conn = Mock()
    mock_cursor = Mock()
    mock_cursor.fetchone.side_effect = [
        (1, 'customer@example.com', 'premium', '101,102', '4242424242424242'),
        (25.00,),
        (15.00,)
    ]
    mock_conn.cursor.return_value = mock_cursor
    monkeypatch.setattr('sqlite3.connect', lambda x: mock_conn)
    
    # Mock Stripe
    mock_charge = Mock()
    mock_charge.id = 'ch_123'
    mock_stripe = Mock()
    mock_stripe.Charge.create.return_value = mock_charge
    monkeypatch.setattr('stripe.Charge', mock_stripe.Charge)
    
    # Mock email
    mock_smtp = Mock()
    monkeypatch.setattr('smtplib.SMTP', lambda x, y: mock_smtp)
    
    # Now we can test!
    processor = OrderProcessor()
    result = processor.process_order(1)
    
    assert result == 'ch_123'
    # Verify premium discount was applied: (25 + 15) * 0.9 = 36
    mock_stripe.Charge.create.assert_called_once()
    assert mock_stripe.Charge.create.call_args[1]['amount'] == 36.0
```

**Step 2: Extract Dependencies**

```python
class OrderProcessor:
    def __init__(self, db=None, payment_gateway=None, email_service=None):
        self.db = db or DatabaseConnection()
        self.payment = payment_gateway or StripeGateway()
        self.email = email_service or EmailService()
    
    def process_order(self, order_id):
        order = self.db.get_order(order_id)
        total = self._calculate_total(order)
        charge_id = self.payment.charge(order.card, total)
        self.email.send_confirmation(order.email, total)
        return charge_id
    
    def _calculate_total(self, order):
        items = self.db.get_items(order.item_ids)
        subtotal = sum(item.price for item in items)
        return self._apply_discount(subtotal, order.customer_type)
    
    def _apply_discount(self, amount, customer_type):
        if customer_type == 'premium':
            return amount * 0.9
        return amount
```

**Step 3: Add Proper Unit Tests**

```python
def test_calculate_total_premium_customer():
    mock_db = Mock()
    mock_db.get_items.return_value = [
        Mock(price=25.00),
        Mock(price=15.00)
    ]
    
    processor = OrderProcessor(db=mock_db)
    order = Mock(item_ids=[101, 102], customer_type='premium')
    
    total = processor._calculate_total(order)
    
    assert total == 36.0  # (25 + 15) * 0.9

def test_apply_discount_premium():
    processor = OrderProcessor()
    result = processor._apply_discount(100, 'premium')
    assert result == 90.0

def test_apply_discount_regular():
    processor = OrderProcessor()
    result = processor._apply_discount(100, 'regular')
    assert result == 100.0
```

## Key Resources

**Books specifically on legacy code:**

- **“Working Effectively with Legacy Code” by Michael Feathers** - THE definitive guide
- **“Refactoring: Improving the Design of Existing Code” by Martin Fowler** - Essential refactoring catalog

**Practice Repositories:**

- **emilybache/GildedRose-Refactoring-Kata** - Famous kata for practicing legacy code refactoring
- **emilybache/Tennis-Refactoring-Kata** - Another excellent practice kata
- **Look for “approval testing” examples on GitHub**

**Courses:**

- Search Pluralsight for “Working with Legacy Code” or “Refactoring Python”
- Emily Bache’s courses on testing and refactoring

## The Mindset Shift

Refactoring legacy code requires patience and discipline. Remember:

- **Make small, incremental changes** - Don’t try to fix everything at once
- **Always have tests before refactoring** - Even characterization tests are better than nothing
- **Refactor to patterns** - Extract method, introduce parameter, dependency injection
- **Leave code better than you found it** - The Boy Scout Rule
- **Celebrate small wins** - Every function you make testable is progress

The goal isn’t perfection—it’s making the code slightly more maintainable with each change, until eventually you’ve transformed a legacy mess into a well-tested, modular system.

Would you like me to elaborate on any specific technique, or would you like a hands-on exercise to practice these concepts?​​​​​​​​​​​​​​​​
