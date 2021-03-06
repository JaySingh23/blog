title: Productionizing A Flask Application
date: 2014-01-29 09:54
categories: python flask bull

When I released [bull](http://www.jeffknupp.com/blog/2014/01/18/python-and-flask-are-ridiculously-powerful/)
as an open source project, it was in quite a state. Everything was in a single
file, there was inline HTML (ew), and both tests and documentation were
non-existent. Over the past week, I've spent some time "productionizing"
`bull`, and recounting the steps I took will likely be helpful to others 
looking to deploy a Flask app to production. In this article, you'll learn how
to organize a Flask application, add testing and documentation, and even how to
enable authentication for "admin-only" content.

<!--more-->

## `bull` looks like a pile of...

The first `git push` of `bull` was a crazy mess, *but it worked*, and that's all
I was concerned with at the time. I knew I would clean everything up "later", so
I wasn't worried about the quality at that time. Besides, anyone capable of
using `bull` in that state was certainly capable of cleaning it up a bit on
their own, if they so desired.

To make it more accessible, however, it needed an overhaul. By focusing
on a few key areas, I was able to make `bull` a solid, production-ready
application. Those areas included:

1. Project layout
1. An "admin" work flow with restricted pages
1. Automated testing
1. Automated documentation generation

I'll discuss each of these sections in detail, as I'm convinced that, if you get
these areas right, you're 90% of the way to having a production-ready
application.

## Everything in its place

`bull` was comprised of a single `app.py` file with all code, templates, and
database models. The first step was simple: **organize the code along MVC lines.**
That meant the models got their own file (`models.py`), the controllers/application
logic got a file (it became `bull.py`), and the views/templates were moved into
a separate directory and implemented as proper Jinja2 templates (in the
`templates` directory, the default location Flask looks for template files).

Here was the contents of `models.py`:

    #!py
    """Database models for the Bull application."""

    import datetime

    from flask.ext.sqlalchemy import SQLAlchemy

    db = SQLAlchemy()

    class Product(db.Model):
        """A digital product for sale on our site.

        :param int id: Unique id for this product
        :param str name: Human-readable name of this product
        :param str file_name: Path to file this digital product represents
        :param str version: Optional version to track updates to products
        :param bool is_active: Used to denote if a product should be considered for-sale
        :param float price: Price of product

        """
        __tablename__ = 'product'
        id = db.Column(db.Integer, primary_key=True, autoincrement=True)
        name = db.Column(db.String)
        file_name = db.Column(db.String)
        version = db.Column(db.String, default=None, nullable=True)
        is_active = db.Column(db.Boolean, default=True, nullable=True)
        price = db.Column(db.Float)

        def __str__(self):
            """Return the string representation of a product."""
            if self.version is not None:
                return '{} (v{})'.format(self.name, self.version)
            return self.name

    class Purchase(db.Model):
        """Contains information about the sale of a product.

        :param str uuid: Unique ID (and URL) generated for the customer unique to this purchase
        :param str email: Customer's email address
        :param int product_id: ID of the product associated with this sale
        :param product: The associated product
        :param downloads_left int: Number of downloads remaining using this URL

        """
        __tablename__ = 'purchase'
        uuid = db.Column(db.String, primary_key=True)
        email = db.Column(db.String)
        product_id = db.Column(db.Integer, db.ForeignKey('product.id'))
        product = db.relationship(Product)
        downloads_left = db.Column(db.Integer, default=5)
        sold_at = db.Column(db.DateTime, default=datetime.datetime.now)

        def sell_date(self):
            return self.sold_at.date()

        def __str__(self):
            """Return the string representation of the purchase."""
            return '{} bought by {}'.format(self.product.name, self.email)

You'll notice that there's a lot of documentation/docstrings in there, and
that's another part of the production-puzzle. Adding documentation that Sphinx
will be able to make sense of and use to generate pretty HTML/PDF output is key.
Obviously, writing the documentation as you go is easier and more productive
than retro-fitting existing code with documentation, but I had to do a bit of
the latter here.

`bull.py` contained all of the "controller" logic for the application. It looked
like this:

    #!py
    """Bull is a library used to sell digital products on your website. It's meant
    to be run on the same domain as your sales page, making analytics tracking
    trivially easy.
    """

    import logging
    import sys
    import uuid
    from collections import defaultdict

    from flask import (Blueprint, send_from_directory, abort, request,
                    render_template, current_app, render_template, redirect,
                    url_for)
    from flask.ext.sqlalchemy import SQLAlchemy
    from flask.ext.mail import Mail, Message
    import stripe

    from .models import Product, Purchase, User, db

    logger = logging.getLogger(__name__)
    bull = Blueprint('bull', __name__)
    mail = Mail()

    @bull.route('/<purchase_uuid>')
    def download_file(purchase_uuid):
        """Serve the file associated with the purchase whose ID is *purchase_uuid*.

        :param str purchase_uuid: Primary key of the purchase whose file we need
                                to serve

        """
        purchase = Purchase.query.get(purchase_uuid)
        if purchase:
            purchase.downloads_left -= 1
            if purchase.downloads_left <= 0:
                return render_template('downloads_exceeded.html')
            db.session.commit()
            return send_from_directory(
                    directory=current_app.config['FILE_DIRECTORY'],
                    filename=purchase.product.file_name,
                    as_attachment=True)
        else:
            abort(404)


    @bull.route('/buy', methods=['POST'])
    def buy():
        """Facilitate the purchase of a product."""

        stripe_token = request.form['stripeToken']
        email = request.form['stripeEmail']
        product_id = request.form['product_id']

        product = Product.query.get(product_id)
        amount = int(product.price * 100)
        try:
            charge = stripe.Charge.create(
                    amount=amount,
                    currency='usd',
                    card=stripe_token,
                    description=email)
        except stripe.CardError:
            return render_template('charge_error.html')

        current_app.logger.info(charge)

        purchase = Purchase(uuid=str(uuid.uuid4()),
                email=email,
                product=product)
        db.session.add(purchase)
        db.session.commit()

        mail_html = render_template(
                'email.html',
                url=purchase.uuid,
                )

        message = Message(
                html=mail_html,
                subject=current_app.config['MAIL_SUBJECT'],
                sender=current_app.config['MAIL_FROM'],
                recipients=[email])

        with mail.connect() as conn:
            conn.send(message)

        return render_template('success.html', url=str(purchase.uuid), purchase=purchase, product=product,
                amount=amount)

    @bull.route('/reports')
    def reports():
        """Run and display various analytics reports."""
        products = Product.query.all()
        purchases = Purchase.query.all()
        purchases_by_day = defaultdict(lambda: {'units': 0, 'sales': 0.0})
        for purchase in purchases:
            purchase_date = purchase.sold_at.date().strftime('%m-%d')
            purchases_by_day[purchase_date]['units'] += 1
            purchases_by_day[purchase_date]['sales'] += purchase.product.price
        purchase_days = sorted(purchases_by_day.keys())
        units = len(purchases)
        total_sales = sum([p.product.price for p in purchases])

        return render_template(
                'reports.html',
                products=products,
                purchase_days=purchase_days,
                purchases=purchases,
                purchases_by_day=purchases_by_day,
                units=units,
                total_sales=total_sales)

    @bull.route('/test/<product_id>')
    def test(product_id):
        """Return a test page for live testing the "purchase" button.

        :param int product_id: id (primary key) of product to test.
        """
        test_product = Product.query.get(product_id)
        return render_template(
                'test.html',
                test_product=test_product)

You'll notice that `bull` is a `Blueprint` rather than a "normal" Flask 
application. This allows `bull` to be added to existing Flask applications
without disruption (a `Blueprint` in Flask is a "pattern" for creating mini, application-like
things like `bull`). You may also notice that there's an endpoint that wasn't present in
the original version: `/reports`. I wanted to enable simple analytics in `bull`, and 
that's what the `/reports` endpoint represents.

## Lock-down

At this point, you may be thinking, "but can't *anyone* go to the `/reports` endpoint and see your
sales numbers?" Yep. And that obviously won't do. What we need is a way to allow only authorized
users to hit that endpoint. This means we'll need to create a user model and deal with all sorts 
of nasty things like a sign-up work-flow, password generation and storage (easy to get wrong), 
and forms. In the interest of me doing as little work as possible, I made use of some of Flask's 
great *extensions*.

I decided to use Flask-Login for authorization. It gives you a `@login_required` decorator you can 
toss in front of sensitive endpoints. It doesn't handle, however, registration. 

Knowing that registration can be a bit of a rabbit hole (and, again, wanting to minimize the amount 
of effort I put into this), I decided that, rather than have a web-based registration work-flow, I would
simply include a script to create an admin user, since in almost all cases a single admin user would suffice.
That meant, however, creating a `User` model and making some changes to the 
application logic. The final `models.py` became the following:


    #!py
    """Database models for the Bull application."""

    import datetime

    from flask.ext.sqlalchemy import SQLAlchemy

    db = SQLAlchemy()

    class Product(db.Model):
        """A digital product for sale on our site.

        :param int id: Unique id for this product
        :param str name: Human-readable name of this product
        :param str file_name: Path to file this digital product represents
        :param str version: Optional version to track updates to products
        :param bool is_active: Used to denote if a product should be considered for-sale
        :param float price: Price of product

        """
        __tablename__ = 'product'
        id = db.Column(db.Integer, primary_key=True, autoincrement=True)
        name = db.Column(db.String)
        file_name = db.Column(db.String)
        version = db.Column(db.String, default=None, nullable=True)
        is_active = db.Column(db.Boolean, default=True, nullable=True)
        price = db.Column(db.Float)

        def __str__(self):
            """Return the string representation of a product."""
            if self.version is not None:
                return '{} (v{})'.format(self.name, self.version)
            return self.name

    class Purchase(db.Model):
        """Contains information about the sale of a product.

        :param str uuid: Unique ID (and URL) generated for the customer unique to this purchase
        :param str email: Customer's email address
        :param int product_id: ID of the product associated with this sale
        :param product: The associated product
        :param downloads_left int: Number of downloads remaining using this URL

        """
        __tablename__ = 'purchase'
        uuid = db.Column(db.String, primary_key=True)
        email = db.Column(db.String)
        product_id = db.Column(db.Integer, db.ForeignKey('product.id'))
        product = db.relationship(Product)
        downloads_left = db.Column(db.Integer, default=5)
        sold_at = db.Column(db.DateTime, default=datetime.datetime.now)

        def sell_date(self):
            return self.sold_at.date()

        def __str__(self):
            """Return the string representation of the purchase."""
            return '{} bought by {}'.format(self.product.name, self.email)

    class User(db.Model):
        """An admin user capable of viewing reports.

        :param str email: email address of user
        :param str password: encrypted password for the user

        """
        __tablename__ = 'user'

        email = db.Column(db.String, primary_key=True)
        password = db.Column(db.String)
        authenticated = db.Column(db.Boolean, default=False)

        def is_active(self):
            """True, as all users are active."""
            return True

        def get_id(self):
            """Return the email address to satisfy Flask-Login's requirements."""
            return self.email

        def is_authenticated(self):
            """Return True if the user is authenticated."""
            return self.authenticated

        def is_anonymous(self):
            """False, as anonymous users aren't supported."""
            return False

The methods `is_active`, `get_id`, `is_authenticated`, and `is_anonymous` are required
by Flask-login and are quite straightforward for our purposes. `User.authenticated` represents
whether or not the user is *currently* authenticated (and thus changes after login/logout).

The changes to `bull.py` were a bit more involved, but still quite simple. Here's the
final version of that file:

    #!py
    """Bull is a library used to sell digital products on your website. It's meant
    to be run on the same domain as your sales page, making analytics tracking
    trivially easy.
    """

    import logging
    import sys
    import uuid
    from collections import defaultdict

    from flask import (Blueprint, send_from_directory, abort, request,
                    render_template, current_app, render_template, redirect,
                    url_for)
    from flaskext.bcrypt import Bcrypt
    from flask.ext.sqlalchemy import SQLAlchemy
    from flask.ext.login import LoginManager, login_required, login_user, logout_user, current_user
    from flask.ext.mail import Mail, Message
    from flask_wtf import Form
    from wtforms import TextField, PasswordField
    from wtforms.validators import DataRequired
    import stripe

    from .models import Product, Purchase, User, db

    logger = logging.getLogger(__name__)
    bull = Blueprint('bull', __name__)
    mail = Mail()
    login_manager = LoginManager()
    bcrypt = Bcrypt()

    class LoginForm(Form):
        """Form class for user login."""
        email = TextField('email', validators=[DataRequired()])
        password = PasswordField('password', validators=[DataRequired()]) 

    @login_manager.user_loader
    def user_loader(user_id):
        """Given *user_id*, return the associated User object.

        :param unicode user_id: user_id (email) user to retrieve
        """
        return User.query.get(user_id)

    @bull.route("/login", methods=["GET", "POST"])
    def login():
        """For GET requests, display the login form. For POSTS, login the current user
        by processing the form."""
        form = LoginForm()
        if form.validate_on_submit():
            user = User.query.get(form.email.data)
            if user and bcrypt.check_password_hash(user.password, form.password.data):
                    user.authenticated = True
                    db.session.add(user)
                    db.session.commit()
                    login_user(user, remember=True)
                    return redirect(url_for("bull.reports"))
        return render_template("login.html", form=form)

    @bull.route("/logout", methods=["GET"])
    @login_required
    def logout():
        """Logout the current user."""
        user = current_user
        user.authenticated = False
        db.session.add(user)
        db.session.commit()
        logout_user()
        return render_template("logout.html")


    @bull.route('/<purchase_uuid>')
    def download_file(purchase_uuid):
        """Serve the file associated with the purchase whose ID is *purchase_uuid*.

        :param str purchase_uuid: Primary key of the purchase whose file we need
                                to serve

        """
        purchase = Purchase.query.get(purchase_uuid)
        if purchase:
            purchase.downloads_left -= 1
            if purchase.downloads_left <= 0:
                return render_template('downloads_exceeded.html')
            db.session.commit()
            return send_from_directory(
                    directory=current_app.config['FILE_DIRECTORY'],
                    filename=purchase.product.file_name,
                    as_attachment=True)
        else:
            abort(404)


    @bull.route('/buy', methods=['POST'])
    def buy():
        """Facilitate the purchase of a product."""

        stripe_token = request.form['stripeToken']
        email = request.form['stripeEmail']
        product_id = request.form['product_id']

        product = Product.query.get(product_id)
        amount = int(product.price * 100)
        try:
            charge = stripe.Charge.create(
                    amount=amount,
                    currency='usd',
                    card=stripe_token,
                    description=email)
        except stripe.CardError:
            return render_template('charge_error.html')

        current_app.logger.info(charge)

        purchase = Purchase(uuid=str(uuid.uuid4()),
                email=email,
                product=product)
        db.session.add(purchase)
        db.session.commit()

        mail_html = render_template(
                'email.html',
                url=purchase.uuid,
                )

        message = Message(
                html=mail_html,
                subject=current_app.config['MAIL_SUBJECT'],
                sender=current_app.config['MAIL_FROM'],
                recipients=[email])

        with mail.connect() as conn:
            conn.send(message)

        return render_template('success.html', url=str(purchase.uuid), purchase=purchase, product=product,
                amount=amount)

    @bull.route('/reports')
    @login_required
    def reports():
        """Run and display various analytics reports."""
        products = Product.query.all()
        purchases = Purchase.query.all()
        purchases_by_day = defaultdict(lambda: {'units': 0, 'sales': 0.0})
        for purchase in purchases:
            purchase_date = purchase.sold_at.date().strftime('%m-%d')
            purchases_by_day[purchase_date]['units'] += 1
            purchases_by_day[purchase_date]['sales'] += purchase.product.price
        purchase_days = sorted(purchases_by_day.keys())
        units = len(purchases)
        total_sales = sum([p.product.price for p in purchases])

        return render_template(
                'reports.html',
                products=products,
                purchase_days=purchase_days,
                purchases=purchases,
                purchases_by_day=purchases_by_day,
                units=units,
                total_sales=total_sales)

    @bull.route('/test/<product_id>')
    def test(product_id):
        """Return a test page for live testing the "purchase" button.

        :param int product_id: id (primary key) of product to test.
        """
        test_product = Product.query.get(product_id)
        return render_template(
                'test.html',
                test_product=test_product)

Helpfully, Flask-login gives you access to the current user as `current_user`, allowing 
easy manipulation of the user's login status. The `user_loader` function is again required 
by Flask-login as a way to find a user based on their ID. In our case, that's a simple operation.

For the lone form required (the login form) I used the excellent Flask-WTF (a wrapper around WTForms).
It gives you a programmatic interface to forms, much the same as Django provides. Our form is trivial,
but more complex form-based work-flows are possible.

## "I don't test often, but when I do, I test in production"

The above quote (which I stole from a T-Shirt I saw a co-worker wearing) was `bull`'s previous testing strategy. No more.
Flask goes out of it's way to make testing easy, so we may as well make use of it. The primary way that
we test an application in Flask is to import the Flask `app` instance and use the included `test_client`. The `test_client`
allows us to make HTTP requests against our application easily, as shown in the file below. The following is taken from the
file `test_bull.py` that lives in a `tests` directory at the top level of our project:

    #!py
    """Tests for the Bull digital goods sales application."""

    import datetime
    import unittest
    import uuid
    import os

    from flask import current_app
    from flask.ext.login import LoginManager, login_required, login_user

    from bull import app, mail, bcrypt
    from bull.models import db, User, Product, Purchase

    class BullTestCase(unittest.TestCase):
        """Main test cases for Bull."""

        def setUp(self):
            """Pre-test activities."""
            app.testing = True
            app.config['STRIPE_SECRET_KEY'] = 'foo'
            app.config['STRIPE_PUBLIC_KEY'] = 'bar'
            app.config['SITE_NAME'] = 'www.foo.com'
            app.config['STRIPE_SECRET_KEY'] = 'foo'
            app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///:memory:'
            app.config['WTF_CSRF_ENABLED'] = False
            app.config['FILE_DIRECTORY'] = os.path.abspath(os.path.join(os.path.split(os.path.abspath(__file__))[0], 'files'))
            with app.app_context():
                db.init_app(current_app)
                db.metadata.create_all(db.engine)
                mail.init_app(current_app)
                bcrypt.init_app(current_app)
                self.db = db
                self.app = app.test_client()
                self.purchase_uuid = str(uuid.uuid4())
                product = Product(
                    name='Test Product',
                    file_name='test.txt',
                    price=5.01)
                purchase = Purchase(product=product,
                        email='foo@bar.com',
                        uuid=self.purchase_uuid,
                        sold_at=datetime.datetime(2014, 1, 1, 12, 12, 12))
                user = User(email='admin@foo.com',
                        password=bcrypt.generate_password_hash('password'))
                db.session.add(product)
                db.session.add(purchase)
                db.session.add(user)
                db.session.commit()


        def test_get_test(self):
            """Does hitting the /test endpoint return the proper HTTP code?"""
            response = self.app.get('/test/1')
            assert response.status_code == 200
            assert app.config['STRIPE_PUBLIC_KEY'] in response.data

        def test_get_user(self):
            """Can we retrieve the User instance created in setUp?"""
            with app.app_context():
                user = User.query.get('admin@foo.com')
                assert bcrypt.check_password_hash(user.password, 'password')

        def test_get_product(self):
            """Can we retrieve the Product instance created in setUp?"""
            with app.app_context():
                product = Product.query.get(1)
                assert product is not None
                assert product.name == 'Test Product'

        def test_get_purchase(self):
            """Can we retrieve the Purchase instance created in setUp?"""
            with app.app_context():
                purchase = Purchase.query.get(self.purchase_uuid)
                assert purchase is not None
                assert purchase.product.price == 5.01
                assert purchase.email == 'foo@bar.com'

        def test_download_file(self):
            """Given an existing purchase, does visiting the purchase's url allow us
            to download the file?."""
            purchase_url = '/' + self.purchase_uuid
            response = self.app.get(purchase_url)
            assert response.data == 'Test content\n'
            assert response.status_code == 200

        def test_product_no_version_as_string(self):
            """Is the string representation of the Product model what we expect?"""
            with app.app_context():
                product = Product.query.get(1)
                assert str(product) == 'Test Product'

        def test_product_with_version_as_string(self):
            """Is the string representation of the Product model what we expect?"""
            with app.app_context():
                product = Product.query.get(1)
                product.version = '1.0'
                assert str(product) == 'Test Product (v1.0)'

        def test_get_purchase_date(self):
            """Can we retrieve the date of the Purchase instance created in setUp?"""
            with app.app_context():
                purchase = Purchase.query.get(self.purchase_uuid)
                assert purchase.sell_date() == datetime.datetime(2014, 1, 1).date()

        def test_get_purchase_string(self):
            """Is the string representation of the Purchase model what we expect?"""
            with app.app_context():
                purchase = Purchase.query.get(self.purchase_uuid)
                assert str(purchase) == 'Test Product bought by foo@bar.com'

        def login(self, username, password):
            """Login user."""
            return self.app.post(
                    '/login', 
                    data={'email': username, 'password': password},
                    follow_redirects=True
                    )

        def test_user_authentication(self):
            """Do the authentication methods for the User model work as expected?"""
            with app.app_context():
                user = User.query.get('admin@foo.com')
                response = self.app.get('/reports')
                assert response.status_code == 401
                assert self.login(user.email, 'password').status_code == 200
                response = self.app.get('/reports')
                assert response.status_code == 200
                assert 'drawSalesChart' in response.data
                response = self.app.get('/logout')
                assert response.status_code == 200
                response = self.app.get('/reports')
                assert response.status_code == 401

The tests are short but rather exhaustive. We set up the test to use an in-memory SQLite database and
add a `Product`, `Purchase`, and `User` object. The tests cover the major functionality of 
the application, with the most complex being the authentication test (though even 
that test is simple compared to the tests of other applications). Notice that most test cases
check both the `status_code` *and* the `data`. Checking one or the other usually is not sufficient.
Notice that, in the `login` method, we're even able to instruct the `test_client` to follow redirects
to emulate our login flow. Testing Flask applications is well covered in the official Flask documentation,
so head there if any of this is confusing.

You may notice I'm using `assert` statements rather than the `unittest` module's `assertTrue` and friends.
That's because I exercise my tests using `py.test` rather than the `unittest` test-runner. I much
prefer the former, but I usually write my tests in a way that is as compatible with `unittest` as possible
in case I decide later to switch to another testing framework. One last thing to note is the docstrings in my
test methods. I've lately been writing test docstrings in the form of a question. I've found that when a test
fails, having the docstring represent the question we're trying to answer makes understanding what the purpose
of an individual test is much more clear.

## Sphinx on steroids

You're probably familiar with Sphinx and its `apidoc` capabilities. By running `sphinx-apidoc`, Sphinx generates
the appropriate `rst` files with `automodule` directives, essentially generating all of the documentation
for you project automatically (without you needing to hand-write `rst` files). Planning to use this in advance
is crucial, as it means you'll be formatting your docstrings in a way that Sphinx recognizes.

What you may *not* be aware of is the existence of a third-party package, `sphinxcontrib-httpdomain`, that 
automatically documents your *HTTP endpoints*. This is a huge win, since for Flask applications, documentation
on the functions that implement the endpoints is usually not what the user is looking for. Rather, they want
to see how to use the endpoints themselves. By adding `sphinxcontrib-httpdomain` to your `conf.py` file and adding
the following directive, you'll get exactly that:

    #!rst
    .. autoflask:: bull:app
        :undoc-static:

This adds nice looking, JSON parameter-aware documentation generation to your project and is something your users
will love you for using.

## Automate all the things

As outlined in my article [Open Sourcing a Python Project the Right Way](http://www.jeffknupp.com/blog/2013/08/16/open-sourcing-a-python-project-the-right-way/),
I set up TravisCI and coveralls.io integration with `bull`, as well as `git-flow` for the branching model. I also added a script, located
at `scripts/bull`, that is installed along with the package and supports a single command, `setup`. Running `bull setup` creates
the requisite `app.py` and `config.py` files as well as the `files` directory. Previously, the user would have to do this manually,
going into their `site-packages` folder and copying the included versions into a new workspace. That was a silly and error-prone work-flow,
so automating it makes sense. The last piece of the puzzle is the `create_user.py` script that populates the database with a 
single `User` object. The code for that file is as follows:


    #!py
    #!/usr/bin/env python
    """Create a new admin user able to view the /reports endpoint."""
    from getpass import getpass
    import sys

    from flask import current_app
    from bull import app, Product, Purchase, bcrypt
    from bull.models import User, db

    def main():
        """Main entry point for script."""
        with app.app_context():
            db.metadata.create_all(db.engine)
            if User.query.all():
                print 'A user already exists! Create another? (y/n):',
                create = raw_input()
                if create == 'n':
                    return

            print 'Enter email address: ',
            email = raw_input()
            password = getpass()
            assert password == getpass('Password (again):')

            user = User(email=email, password=bcrypt.generate_password_hash(password))
            db.session.add(user)
            db.session.commit()
            print 'User added.'


    if __name__ == '__main__':
        sys.exit(main())

It's straightforward and does only what it needs to create a new user. Since it only needs to be run
once per installation, I'm not too worried about adding bells and whistles.

One other nice piece of automation is a script I wrote for [sandman](http://www.sandman.io): `update_version.sh`.
It automatically does the following:

1. starts a release in `git-flow`
1. updates the `__version__` string in the package's `__init__.py` file
1. deletes and re-generates the documentation 
1. commits the `__init__.py` change
1. finishes the `git-flow` release
1. uploads the new package to PyPI
1. uploads the new documentation to pythonhosted.org

For those interested, here are the contents of the script, though you can probably guess them from the list above:

    #!bash
    git flow release start v$1
    sed -i -e "s/__version__ = '.*'/__version__ = '$1'/g" bull/__init__.py
    rm -rf docs/generated
    python setup.py develop
    make docs
    git commit docs bull/__init__.py -m "Update to version v$1"
    git flow release finish v$1
    python setup.py sdist upload -r pypi
    python setup.py upload_docs -r pypi

## Wrapping up

So that's about it. From the mess of an application that the original `bull` release was, I've
gotten `bull` to a place I'm happy with. The work-flow is all automated and includes sufficient
testing and documentation. Using `bull` is simply a matter of `pip install`ing it, running `bull setup`,
adding your configuration values, then configuring your web server to run it. That's all that's required
to have a self-hosted digital goods payment solution with integrated analytics, and I'm pretty happy about 
that.
