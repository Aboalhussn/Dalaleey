# Dalaleey

from flask import Flask, redirect , render_template , request ,url_for , session, flash, send_from_directory
from flask_sqlalchemy import SQLAlchemy
from sqlalchemy import  Column, Integer, String, ForeignKey, Float, DateTime, Text, Boolean
from sqlalchemy.orm import relationship
from datetime import datetime
from werkzeug.security import generate_password_hash, check_password_hash
from werkzeug.utils import secure_filename
import os

app = Flask(__name__)
Usertype = ["Tenant","Agent","Investor"]


UPLOAD_FOLDER = 'uploads'
app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER
ALLOWED_EXTENSIONS = {'png', 'jpg', 'jpeg', 'gif'}


app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///dalaley.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

app.config["SECRET_KEY"] = 'nbvuiwuibnviubwuivbgquipbvubcuivbevyubdh'


db = SQLAlchemy(app)

class User(db.Model):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String(100), nullable=False)
    email = Column(String(100), unique=True, nullable=False)
    password = Column(String(255), nullable=False)
    user_type = Column(String(50), nullable=False)  # "owner" أو "tenant"
    created_at = Column(DateTime, default=datetime.utcnow)


    properties = relationship("Property", back_populates="owner")
    transactions = relationship("Transaction", back_populates="user")
    favorites = relationship("Favorite", back_populates="user")


class Property(db.Model):
    __tablename__ = 'properties'

    id = Column(Integer, primary_key=True, autoincrement=True)
    title = Column(String(200), nullable=False)
    description = Column(Text)
    price = Column(Float, nullable=False)
    property_type = Column(String(50), nullable=False)  # "residential" أو "commercial"
    is_available = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)


    user_id = Column(Integer, ForeignKey('users.id'), nullable=False)
    owner = relationship("User", back_populates="properties")

    images = relationship("PropertyImage", back_populates="property")
    transactions = relationship("Transaction", back_populates="property")

    favorites = relationship("Favorite", back_populates="property")

class Favorite(db.Model):
    __tablename__ = 'favorites'

    id = Column(Integer, primary_key=True, autoincrement=True)
    user_id = Column(Integer, ForeignKey('users.id'), nullable=False)
    property_id = Column(Integer, ForeignKey('properties.id'), nullable=False)


    user = relationship("User", back_populates="favorites")
    property = relationship("Property", back_populates="favorites")

class PropertyImage(db.Model):
    __tablename__ = 'property_images'

    id = Column(Integer, primary_key=True, autoincrement=True)
    property_id = Column(Integer, ForeignKey('properties.id'), nullable=False)
    image_url = Column(String(255), nullable=False)


    property = relationship("Property", back_populates="images")


class Transaction(db.Model):
    __tablename__ = 'transactions'

    id = Column(Integer, primary_key=True, autoincrement=True)
    user_id = Column(Integer, ForeignKey('users.id'), nullable=False)
    property_id = Column(Integer, ForeignKey('properties.id'), nullable=False)
    transaction_type = Column(String(50), nullable=False)  # "rent" أو "purchase"
    transaction_date = Column(DateTime, default=datetime.utcnow)


    user = relationship("User", back_populates="transactions")
    property = relationship("Property", back_populates="transactions")


with app.app_context():
    db.create_all()
    print("Database created")


@app.route("/")
def index():
    return render_template("index.html")


@app.route("/register", methods=["POST", "GET"])
def register():

    if request.method == "POST":

        email = request.form.get("email")
        password = request.form.get("password")
        name = request.form.get("username")
        usertype = request.form.get("userType")


        if not email:
            flash('Enter your Email!', category='error')
        elif not password:
            flash('Enter the password!!', category='error')
        elif not name:
            print("12345")
        elif not usertype:
            flash('Choes your type', category='error')
        else:
            flash("Welcome in Dalaley")


        existing_user = User.query.filter_by(email=email).first()
        if existing_user:
            flash('you have account' , category='error')


        hashed_password = generate_password_hash(password)


        new_user = User(name=name, email=email, password=hashed_password, user_type=usertype)
        db.session.add(new_user)
        db.session.commit()


    return render_template('register.html')



@app.route("/login", methods = ["POST", "GET"])
def login():
    if request.method == "POST":
        email = request.form.get("email")
        password = request.form.get("password")


        if not email or not password:
            return render_template("login.html", message="Email and password are required.")


        user = User.query.filter_by(email=email).first()
        if not user:
            return render_template("login.html", message="User  does not exist.")


        if not check_password_hash(user.password, password):
            return render_template("login.html", message="Invalid password.")


        print(user.id)
        session['id'] = user.id
        session['name'] = user.name


        return redirect("/")

    return render_template("login.html")



@app.route("/profile")
def profile():
    return render_template("profile.html", user=User)


@app.route("/logout")
def logout():

    return render_template("logoutconf.html")

@app.route("/logout_page")
def logout_page():


    session.pop('user_id', None)
    session.pop('user_name', None)


    session.clear()

    return render_template('logout.html')


@app.route("/help")
def help():
    return render_template("/help.html")

@app.route("/about")
def about():
    return render_template("about.html")


@app.route("/property/<pid>", methods = ["Get"])
def getProperty(pid):
    prop = Property.query.join(PropertyImage).first(pid)
    if(not prop):
        render_template("error.html")

    return render_template("property.html", property=prop)


@app.route("/property", methods=["POST","GET"])
def createProperty():
    if 'id' not in session:
        return redirect(url_for("login"))

    if request.method == "POST":
        title = request.form.get("title")
        description = request.form.get("description")
        price = request.form.get("price")
        property_type = request.form.get("property_type")
        is_available = request.form.get("is_available") == 'on'


        if not title or not description or not price or not property_type:
            return "يرجى ملء جميع الحقول المطلوبة", 400

        prop = Property(
            title=title,
            description=description,
            price=price,
            property_type=property_type,
            is_available=is_available,
            created_at=datetime.utcnow(),
            user_id=session['id']
        )

        images = request.files.getlist("images")
        for img_file in images:
            if allowed_file(img_file.filename):
                filepath = os.path.join("static/images", img_file.filename)
                img_file.save(filepath)
                img = PropertyImage(image_url=filepath)
                prop.images.append(img)
            else:
                pass


        db.session.add(prop)
        db.session.commit()

        return redirect(url_for("properties"))


    return render_template("add_property.html")

def allowed_file(filename):
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS

@app.route('/upload', methods=['GET', 'POST'])
def upload_file():
    if request.method == 'POST':

        if 'file' not in request.files:
            return render_template("error.html", message="No file part")
        file = request.files['file']


        if file.filename == '':
            return render_template("error.html", message="No selected file")


        if file and allowed_file(file.filename):
            filename = secure_filename(file.filename)
            filepath = os.path.join(app.config['UPLOAD_FOLDER'], filename)


            file.save(filepath)


            new_image = PropertyImage(property_id=1, image_url=filename)
            db.session.add(new_image)
            db.session.commit()


            return redirect(url_for('uploaded_file', filename=filename))
        else:
            return render_template("error.html", message="File type not allowed")

    return render_template('upload.html')

@app.route('/favorite/<int:property_id>', methods=['POST'])
def favorite(property_id):
    if 'id' not in session:
        return redirect(url_for('login'))

    user_id = session['id']
    existing_favorite = Favorite.query.filter_by(user_id=user_id, property_id=property_id).first()

    if existing_favorite:
        return render_template("error.html", message="Property already favorited.")

    new_favorite = Favorite(user_id=user_id, property_id=property_id)
    db.session.add(new_favorite)
    db.session.commit()

    return redirect(url_for('index'))

@app.route("/notification")
def notification():
    return render_template("notification.html")

@app.route('/favorites')
def favorites():
    if 'id' not in session:
        return redirect(url_for('login'))

    user_id = session['id']
    favorite_properties = Favorite.query.filter_by(user_id=user_id).all()
    properties = [favorite.property for favorite in favorite_properties]

    return render_template('favorites.html', properties=properties)

@app.route("/properties")
def properties():
    all_properties = Property.query.all()
    print(all_properties)
    return render_template("properties.html", properties=all_properties)


@app.route("/error")
def error():
    return render_template("error.html")

if __name__=="__main__":
    app.run(debug=True)
