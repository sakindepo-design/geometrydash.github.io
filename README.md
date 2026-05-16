#include <SFML/Graphics.hpp>
#include <vector>
#include <cstdlib>
#include <ctime>

// Oyun Sabitleri
const int GENISLIK = 800;
const int YUKSEKLIK = 500;
const float ZEMIN_Y = 400.0f;
const float YER_CEKIMI = 0.6f;
const float ZIPLAMA_GUCU = -12.0f;

// 1. OYUNCU (KÜP) SINIFI
class Kup {
public:
    sf::RectangleShape sekil;
    float hizY;
    bool yerde;
    float aci;

    Kup() {
        sekil.setSize(sf::Vector2f(40.0f, 40.0f));
        sekil.setFillColor(sf::Color(0, 255, 200)); // Turkuaz
        sekil.setOutlineThickness(3.0f);
        sekil.setOutlineColor(sf::Color::White);
        // Dönme hareketinin merkezden olması için orijini ortaya alıyoruz
        sekil.setOrigin(20.0f, 20.0f); 
        reset();
    }

    void reset() {
        sekil.setPosition(100.0f, ZEMIN_Y - 20.0f);
        hizY = 0.0f;
        yerde = true;
        aci = 0.0f;
        sekil.setRotation(aci);
    }

    void zipla() {
        if (yerde) {
            hizY = ZIPLAMA_GUCU;
            yerde = false;
        }
    }

    void guncelle() {
        hizY += YER_CEKIMI;
        sekil.move(0.0f, hizY);

        // Zemine çarpma kontrolü
        if (sekil.getPosition().y >= ZEMIN_Y - 20.0f) {
            sekil.setPosition(sekil.getPosition().x, ZEMIN_Y - 20.0f);
            hizY = 0.0f;
            yerde = true;
            // Yere bastığında açıyı en yakın 90 dereceye yuvarla
            aci = static_cast<int>(aci + 45) / 90 * 90;
            sekil.setRotation(aci);
        }

        // Havadaysa sürekli dönsün
        if (!yerde) {
            aci += 4.0f;
            sekil.setRotation(aci);
        }
    }
};

// 2. ENGEL (ÜÇGEN) SINIFI
class Engel {
public:
    sf::ConvexShape ucgen;
    float hizX;

    Engel(float baslangicX) {
        hizX = 6.0f;
        ucgen.setPointCount(3);
        // Üçgenin köşelerini tanımlıyoruz (Taban altta, tepe yukarıda)
        ucgen.setPoint(0, sf::Vector2f(0.0f, 0.0f));          // Sol alt
        ucgen.setPoint(1, sf::Vector2f(30.0f, 0.0f));         // Sağ alt
        ucgen.setPoint(2, sf::Vector2f(15.0f, -40.0f));       // Tepe noktası
        
        ucgen.setFillColor(sf::Color(255, 50, 50)); // Kırmızı
        ucgen.setOutlineThickness(2.0f);
        ucgen.setOutlineColor(sf::Color::White);
        ucgen.setPosition(baslangicX, ZEMIN_Y);
    }

    void guncelle() {
        ucgen.move(-hizX, 0.0f);
    }
};

// 3. ANA PROGRAM
int main() {
    std::srand(static_cast<unsigned int>(std::time(nullptr)));

    // Pencere oluşturma
    sf::RenderWindow pencere(sf::VideoMode(GENISLIK, YUKSEKLIK), "Geometry Dash C++");
    pencere.setFramerateLimit(60);

    // Zemin Çizgisi
    sf::RectangleShape zemin(sf::Vector2f(GENISLIK, YUKSEKLIK - ZEMIN_Y));
    zemin.setFillColor(sf::Color(40, 60, 90));
    zemin.setPosition(0.0f, ZEMIN_Y);

    Kup oyuncu;
    std::vector<Engel> engeller;
    
    int engelSayaci = 0;
    int yeniEngelSüresi = 80;
    bool oyunBitti = false;

    // OYUN DÖNGÜSÜ
    while (pencere.isOpen()) {
        sf::Event etkinlik;
        while (pencere.pollEvent(etkinlik)) {
            if (etkinlik.type == sf::Event::Closed)
                pencere.close();

            if (etkinlik.type == sf::Event::KeyPressed) {
                if (etkinlik.key.code == sf::sf::Keyboard::Space) {
                    if (oyunBitti) {
                        // Oyun bittiyse her şeyi sıfırla
                        oyuncu.reset();
                        engeller.clear();
                        engelSayaci = 0;
                        oyunBitti = false;
                    } else {
                        oyuncu.zipla();
                    }
                }
            }
        }

        // OYUN MANTIĞI GÜNCELLEME
        if (!oyunBitti) {
            oyuncu.guncelle();

            // Rastgele aralıklarla yeni engel ekleme
            engelSayaci++;
            if (engelSayaci >= yeniEngelSüresi) {
                engeller.push_back(Engel(GENISLIK + 50.0f));
                engelSayaci = 0;
                yeniEngelSüresi = 70 + std::rand() % 40; // 70 ile 110 frame arası rastgele
            }

            // Engelleri hareket ettir ve çarpışma kontrolü yap
            for (size_t i = 0; i < engeller.size(); i++) {
                engeller[i].guncelle();

                // Basit Kutu (Bounding Box) Çarpışma Testi
                if (oyuncu.sekil.getGlobalBounds().intersects(engeller[i].ucgen.getGlobalBounds())) {
                    oyunBitti = true;
                }
            }

            // Ekrandan çıkan engelleri temizle
            if (!engeller.empty() && engeller[0].ucgen.getPosition().x < -50.0f) {
                engeller.erase(engeller.begin());
            }
        }

        // EKRANA ÇİZME İŞLEMLERİ
        pencere.clear(sf::Color(20, 30, 50)); // Arka plan koyu lacivert

        pencere.draw(zemin);
        pencere.draw(oyuncu.sekil);
        for (const auto& engel : engeller) {
            pencere.draw(engel.ucgen);
        }

        pencere.display();
    }

    return 0;
}
