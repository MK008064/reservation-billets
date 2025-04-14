# app_fixed.py
from flask import Flask, render_template, redirect, url_for, request, flash
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager, UserMixin, login_user, logout_user, login_required, current_user
from werkzeug.security import generate_password_hash, check_password_hash
import os

basedir = os.path.abspath(os.path.dirname(__file__))

app = Flask(__name__)
app.config['SECRET_KEY'] = 'secret'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///' + os.path.join(basedir, 'instance', 'reservation.db')
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

db = SQLAlchemy(app)
login_manager = LoginManager(app)
login_manager.login_view = 'login'

class Utilisateur(UserMixin, db.Model):
    id = db.Column(db.Integer, primary_key=True)
    nom = db.Column(db.String(100))
    email = db.Column(db.String(100), unique=True)
    mot_de_passe = db.Column(db.String(200))

class Evenement(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    titre = db.Column(db.String(200))
    description = db.Column(db.String(300))
    places_disponibles = db.Column(db.Integer)

class Reservation(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    utilisateur_id = db.Column(db.Integer, db.ForeignKey('utilisateur.id'))
    evenement_id = db.Column(db.Integer, db.ForeignKey('evenement.id'))
    statut = db.Column(db.String(50), default='confirmé')
    
    utilisateur = db.relationship('Utilisateur', backref='reservations')
    evenement = db.relationship('Evenement', backref='reservations')

@login_manager.user_loader
def load_user(user_id):
    return Utilisateur.query.get(int(user_id))

@app.route('/')
def index():
    evenements = Evenement.query.all()
    return render_template('index.html', evenements=evenements)

@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        nom = request.form['nom']
        email = request.form['email']
        mot_de_passe = generate_password_hash(request.form['mot_de_passe'])

        if Utilisateur.query.filter_by(email=email).first():
            flash("Cet e-mail est déjà utilisé.")
            return redirect(url_for('register'))

        nouvel_utilisateur = Utilisateur(nom=nom, email=email, mot_de_passe=mot_de_passe)
        db.session.add(nouvel_utilisateur)
        db.session.commit()
        flash("Inscription réussie. Connectez-vous !")
        return redirect(url_for('login'))

    return render_template('register.html')

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        email = request.form['email']
        mot_de_passe = request.form['mot_de_passe']
        utilisateur = Utilisateur.query.filter_by(email=email).first()

        if utilisateur and check_password_hash(utilisateur.mot_de_passe, mot_de_passe):
            login_user(utilisateur)
            return redirect(url_for('index'))
        else:
            flash("Email ou mot de passe incorrect.")
            return redirect(url_for('login'))

    return render_template('login.html')

@app.route('/logout')
@login_required
def logout():
    logout_user()
    return redirect(url_for('index'))

@app.route('/reserver/<int:evenement_id>')
@login_required
def reserver(evenement_id):
    evenement = Evenement.query.get_or_404(evenement_id)
    if evenement.places_disponibles > 0:
        reservation = Reservation(utilisateur_id=current_user.id, evenement_id=evenement.id)
        evenement.places_disponibles -= 1
        db.session.add(reservation)
        db.session.commit()
        flash("Réservation confirmée !")
    else:
        flash("Désolé, plus de places disponibles.")
    return redirect(url_for('index'))

@app.route('/annuler/<int:reservation_id>')
@login_required
def annuler_reservation(reservation_id):
    reservation = Reservation.query.get_or_404(reservation_id)
    if reservation.utilisateur_id == current_user.id and reservation.statut == 'confirmé':
        reservation.statut = 'annulé'
        evenement = Evenement.query.get(reservation.evenement_id)
        evenement.places_disponibles += 1
        db.session.commit()
        flash("Réservation annulée.")
    else:
        flash("Impossible d'annuler cette réservation.")
    return redirect(url_for('mes_reservations'))

@app.route('/mes-reservations')
@login_required
def mes_reservations():
    reservations = Reservation.query.filter_by(utilisateur_id=current_user.id).all()
    return render_template('mes_reservations.html', reservations=reservations)

@app.route('/admin/reservations')
@login_required
def admin_reservations():
    if current_user.email != "admin@site.com":
        flash("Accès refusé.")
        return redirect(url_for('index'))
    reservations = Reservation.query.all()
    return render_template('admin_reservations.html', reservations=reservations)

if __name__ == '__main__':
    if not os.path.exists('instance'):
        os.makedirs('instance')
    with app.app_context():
        db.create_all()
    app.run(debug=True)
