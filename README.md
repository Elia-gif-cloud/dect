from flask import Flask, request, jsonify, render_template, session, redirect, url_for
from flask_sqlalchemy import SQLAlchemy
from werkzeug.security import generate_password_hash, check_password_hash
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity
import requests

app = Flask(__name__)
app.secret_key = 'supersecretkey'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///dec_search.db'
db = SQLAlchemy(app)

# Database Models
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(150), unique=True, nullable=False)
    password = db.Column(db.String(256), nullable=False)
    balance = db.Column(db.Float, default=0.0)
    search_count = db.Column(db.Integer, default=0)
    share_count = db.Column(db.Integer, default=0)

class SearchHistory(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    query = db.Column(db.String(500), nullable=False)
    results = db.Column(db.Text, nullable=False)

# Sample documents
documents = {
    "1": "Artificial Intelligence is the simulation of human intelligence in machines.",
    "2": "Machine Learning is a subset of AI that enables computers to learn from data.",
    "3": "Deep Learning is a more advanced branch of Machine Learning with neural networks.",
    "4": "Natural Language Processing allows computers to understand and process human language.",
    "5": "AI is transforming industries such as healthcare, finance, and education."
}

# Authentication Routes
@app.route('/register', methods=['POST'])
def register():
    data = request.get_json()
    hashed_password = generate_password_hash(data['password'], method='pbkdf2:sha256')
    new_user = User(username=data['username'], password=hashed_password)
    db.session.add(new_user)
    db.session.commit()
    return jsonify({"message": "User registered successfully!"})

@app.route('/login', methods=['POST'])
def login():
    data = request.get_json()
    user = User.query.filter_by(username=data['username']).first()
    if user and check_password_hash(user.password, data['password']):
        session['user_id'] = user.id
        return jsonify({"message": "Login successful!"})
    return jsonify({"error": "Invalid credentials"}), 401

@app.route('/logout')
def logout():
    session.pop('user_id', None)
    return redirect(url_for('home'))

# AI Search System
def search_ai_system(query, docs, top_n=3):
    if not query.strip():
        return "No query entered. Please provide a search term.", []
    
    doc_keys = list(docs.keys())
    corpus = list(docs.values()) + [query]
    
    vectorizer = TfidfVectorizer(stop_words='english')
    tfidf_matrix = vectorizer.fit_transform(corpus)
    
    query_vector = tfidf_matrix[-1]
    doc_vectors = tfidf_matrix[:-1]
    
    similarities = cosine_similarity(query_vector, doc_vectors).flatten()
    top_indices = similarities.argsort()[-top_n:][::-1]
    
    if max(similarities) == 0:
        return "No relevant results found.", []
    
    results = [docs[doc_keys[i]] for i in top_indices]
    return "Search results:", results

@app.route('/')
def home():
    return render_template("index.html")

@app.route('/search', methods=['POST'])
def search():
    if 'user_id' not in session:
        return jsonify({"error": "Please log in to search"}), 401
    
    user = User.query.get(session['user_id'])
    data = request.get_json()
    query = data.get("query", "")
    message, results = search_ai_system(query, documents)
    
    # Store search history
    new_search = SearchHistory(user_id=user.id, query=query, results=str(results))
    db.session.add(new_search)
    db.session.commit()
    
    # Update search count and funding
    user.search_count += 1
    if user.search_count % 10 == 0:
        user.balance += 10
    if user.share_count > 3:
        user.search_count = 5  # Reset search limit
    db.session.commit()
    
    return jsonify({"message": message, "results": results})

@app.route('/share', methods=['POST'])
def share():
    if 'user_id' not in session:
        return jsonify({"error": "Please log in to share"}), 401
    
    user = User.query.get(session['user_id'])
    user.share_count += 1
    if user.share_count > 3:
        user.search_count = 5  # Reset search limit
    db.session.commit()
    return jsonify({"message": "Shared successfully!"})

@app.route('/funding', methods=['POST'])
def funding():
    if 'user_id' not in session:
        return jsonify({"error": "Please log in to fund"}), 401
    
    user = User.query.get(session['user_id'])
    if user.balance >= 50:
        user.balance = 0  # Reset balance after funding
        db.session.commit()
        return jsonify({"message": "Funding processed!"})
    return jsonify({"error": "Insufficient balance for funding"})

if __name__ == '__main__':
    db.create_all()
    app.run(debug=True)
# dect
