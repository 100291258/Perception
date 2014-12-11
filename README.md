Perception
==========

/**
/* @brief Leer un codigo similar al qr.
/* @author Alvaro Muñoz Serrano, Cristina Muñoz Hernandez.
/* @date 6/12/2014
/* @version 1.0
**/

//Bibliotecas.
#include "stdafx.h"
#include <cv.h>
#include <highgui.h>
#include "math.h"
#include <iostream>

using namespace std;
using namespace cv;

//Declaracion de todas las funciones que se va a usar.
double isSquare(const CvPoint* P1, const CvPoint* P2, const CvPoint* P3, const CvPoint* P4, double *angle, double *side);
bool isCode(const Mat *code);
Mat rotateImage(const Mat& source, double angle, Point2f center);
char *readCode(const Mat *code);
char readChar(const Mat* code, int pos);
char readLetter(const Mat* code, int x, int y);

// Esta estructura guarda 4 puntos correspondientes a los vertices de un cuadrado.
struct Square {
   CvPoint P1;
   CvPoint P2;
   CvPoint P3;
   CvPoint P4;
};

int main()
{
// Abrimos el fichero correspondiente y lo almacenamos en img.
  IplImage* img =  cvLoadImage("koala.png");
  
 cvNamedWindow("Raw");
 cvShowImage("Raw",img);

  //Pasamos la imagen a escala de grises.
 IplImage* imgGrayScale = cvCreateImage(cvGetSize(img), 8, 1); 
 cvCvtColor(img,imgGrayScale,CV_BGR2GRAY);

  //Umbralizamos la escala de grises para obtener mejores resultados.
 cvThreshold(imgGrayScale,imgGrayScale,128,255,CV_THRESH_BINARY);  

 CvSeq* contours;  //Puntero para guardar un contorno.
 CvSeq* result;   //Puntero para guardar una secuencia de contornos.
 CvMemStorage *storage = cvCreateMemStorage(0); //lugar para guardar los contornos.
 
//Funcion para encontrar todos los contornos de una imagen.
 cvFindContours(imgGrayScale, storage, &contours, sizeof(CvContour), CV_RETR_LIST, CV_CHAIN_APPROX_SIMPLE, cvPoint(0,0));
  
	Square *squares; //Array para guardar todos los cuadrados que encontremos.
	int size = 0; //Numero de cuadrados encontrados.

	 double maxDiag=0; //Tamaño de la diagonal del cuadrado mas grande encontrado.
	 int bigger; //Posicion en el array del cuadrado mas grande encontrado.
	 double angle; //Angulo con respecto a a la horizontal que esta girado un cuadrado.
	 double side; //Lado del cuadrado.
	 double maxSide = 0; //Lado del cuadrado mas grande.
	 double maxAngle = 0; //Angulo del cuadrado mas grande.

	 //Se itera para cada contorno.
 while(contours)
 {
     //cvApproxPoly obtiene la secuancia de puntos de un contorno (vertices).
     result = cvApproxPoly(contours, sizeof(CvContour), storage, CV_POLY_APPROX_DP, cvContourPerimeter(contours)*0.02, 0);
           
    
	//Si el poligono tiene 4 vertices podria ser un cuadrado.
     if(result->total==4 )
     {
         //Se obtiene cada punto.
         CvPoint *pt[4];
         for(int i=0;i<4;i++){
             pt[i] = (CvPoint*)cvGetSeqElem(result, i);
         }

	// La funcion isSquare devuelve la diagonal si se trata de un cuadrado. En caso contrario devuelve 0. 
	double diag = isSquare(pt[0],pt[1],pt[2],pt[3], &angle, &side);

		//Si es un cuadrado:
		if(diag != 0){
			//Si es el mas grande encontrado hasta ahora se guardan sus datos.
			if (diag>maxDiag){
				//Se guarda la diagonal maxima.
				maxDiag=diag;
				//Se guarda la posicion en el array del cuadrado.
				maxSide = side;
				bigger = size;
				//Se guarda el angulo que esta girado.
				maxAngle = angle;
			}
			// Se guarda el cuadrado en el array:
				// Creamos un array de cuadrados del nuevo tamaño.
			Square *aux = new Square[(size+1)];

			// Guardamos los anteriores cuadrados en ele auxiliar.
			for(int i=0; i<size; i++){
				aux[i] = squares[i];
			}
			// Guardamos el nuevo cuadrado en el auxiliar.
			aux[size].P1 = *pt[0];
			aux[size].P2 = *pt[1];
			aux[size].P3 = *pt[2];
			aux[size].P4 = *pt[3];

			// Borramos el array de cuadrados.
			if(size > 0) delete[] squares;
				//Aumentamos el tamaño.
			size++;
				// Volvemos a crear el array de cuadrados con el nuevo tamaño.
			squares = new Square[size];
				//	Guardamos todos los cuadrados del auxiliar.
			for(int i=0; i<(size); i++){
				squares[i] = aux[i];
			}
				// Borramos el auxiliar.
			if(aux) delete[] aux;
	 } 		 
     }

      //Obtenemos el siguiente contorno.
     contours = contours->h_next; 
 }
 
  //Una vez obtenido el cuadrado mas grande, se recortará y se almacenara en otra imagen.
 CvPoint2D32f punto;
 punto.x = squares[bigger].P1.x;
 punto.y = squares[bigger].P1.y;

  //Se crea un array tipo Mat para poder usar rotateImage.
 Mat mat(img,true);
 
 //Se gira la imagen el angulo calculado anteriormente para que el cuadrado quede recto.
 Mat rotated = rotateImage(mat,maxAngle*360/(2*3.1416),punto);
 
 Mat met;// Aqui se guardara la imagen del cuadrado.
  //Se recorta el cuadrado y se guarda en "met".
 Rect roi(punto.x, punto.y, maxSide, maxSide);
Mat image_roi = rotated(roi);
image_roi.copyTo(met);
imwrite("cropimage.jpg",image_roi);

//Se reescala la imagen a 420x420 para mayor simplicidad mas adelante.
Mat code;
Size code_size(420,420);
resize(image_roi, code, code_size);

//Se comprobara si es un codigo como el que buscamos, y si no lo es se girará la imagen 90 grados.
 //Esto se repetirá 3 veces, ya que no tendria sentido girarlo mas veces.
for(int i = 0; i <=3; i++){
	//isCode comprueba si es o no es un codigo.
	if(isCode(&code)){
		//Si es un codigo, se mostrara el codigo en una ventana y se imprimirá por pantalla su texto.
		cout<<"Codigo: ";
		imshow("code",code);
		readCode(&code);
		break;
	}
	//Si no es un codigo, se girara 90 grados con respecto al centro y se volvera a comprobar.
	else{
		punto.x=210;
		punto.y=210;

		code = rotateImage(code,90,punto);
	}
}

//Cuando el programa acaba, paramos la ejecucion.
 cvWaitKey(0);

 //Liberamos la memoria.
 cvDestroyAllWindows();
 if(squares)delete[] squares;
 cvReleaseMemStorage(&storage);
 cvReleaseImage(&img);
 cvReleaseImage(&imgGrayScale);

  return 0;
}

/**
/* @brief Esta funcion devuelve la distancia entre dos puntos.
/* @param P1, P2 Puntos entre los que se va a calcular la distancia.  
/* @return Distancia entre dos puntos.
**/
double dist(const CvPoint* P1, const CvPoint* P2){
	//Se intenta trabajar con double el menor numero de veces posible para mejorar el tiempo de ejecucion del programa.
	return sqrt((double)((P1->x - P2->x)*(P1->x - P2->x) + (P1->y - P2->y)*(P1->y - P2->y))); 
}


bool areEqual(const CvPoint* P1, const CvPoint* P2, const CvPoint* P3){
	if((P2->x-P1->x)*(P2->x-P1->x)+(P2->y-P1->y)*(P2->y-P1->y) == (P3->x-P1->x)*(P3->x-P1->x) + (P2->y-P1->y)*(P2->y-P1->y)){
		return true;
	}
return false;
}

/**
/* @brief Esta funcion comprueba si los 4 puntos introducidos forman o no un cuadrado y ademas calcula el angulo, el lado y devuelve la diagonal.
/* @param P1, P2, P3, P4 Puntos del cuadrilatero.
/* @param *angle angulo del cuadrado co la horizontal.
/* @param *side lado del cuadrado.
/* @return Diagonal si es un cuadrado, 0 si no lo es.
**/
double isSquare(const CvPoint* P1, const CvPoint* P2, const CvPoint* P3, const CvPoint* P4, double *angle, double *side){
	//Calculamos la distancia entre los puntos contiguos.
	double d01 = dist(P1, P2);
	double d02 = dist(P1, P3);
	double d03 = dist(P1, P4);

	//Hay 3 opciones para que sea un cuadrado: d01=d02 & d01=d03/sqrt(2),  	d01=d03 & d01=d02/sqrt(2)  y  d02=d03 & d02=d01/sqrt(2)
	//Si se cumple alguna de estas se calculara el lado y angulo y se devolvera la diagonal.
	//Hemos decidido dejar un margen de 20 pixeles al encontrar un cuadrado para poder detectar codigos con una camara que no este completamente recta.
		if(d01>=(d02-10) && d01<=(d02+10) && d01>=(d03/1.414213562-10) && d01<=(d03/1.414213562+10)){
			*angle = atan((double)(P1->y - P2->y)/(P1->x - P2->x));
			*side = d01;
				return d03;
		}
		else if(d01>=(d03-10) && d01<=(d03+10) && d01>=(d02/1.414213562-10) && d01<=(d02/1.414213562+10)){
			*angle = atan((double)(P1->y - P2->y)/(P1->x - P2->x));
			*side = d01;
			return d02;
		}
		else if(d02>=(d03-10) && d02<=(d03+10) && d02>=(d01/1.414213562-10) && d02<=(d01/1.414213562+10)){
			*angle = atan((double)(P1->y - P3->y)/(P1->x - P3->x));
			*side = d02;
			return d01;
		}
		return 0;
}

/**
/* @brief Esta funcion comprueba si el cuadrado obtenido corresponde a un codigo o no.
/* @param *code matriz con un posible codigo.
/* @return true si es un codigo, false si no.
**/
bool isCode(const Mat *code){
	
	//Binarizamos y umbralizamos la imagen para obtener mejores resultados.
	IplImage aux(*code);
	cvThreshold(&aux,&aux,128,255,CV_THRESH_BINARY); 	
	IplImage* grayframe = cvCreateImage(cvGetSize(&aux), IPL_DEPTH_8U, 1);
	cvCvtColor(&aux, grayframe, CV_RGB2GRAY);
	Mat mat(grayframe,true);
	
	//Nuestro codigo tiene 21 cuadrados, y las dimensiones de la imagen son 420x420, por lo que cada cuadrado del codigo equivale a 20 pixeles.
	//Para comprobar el color de un cuadrado comprobamos el color de su pixel central. Por ejemplo el primer cuadrado es 10,10, el segundo 30,10 etc.
	//Comprobamos mediante un bucle si los cuadrados de la segunda fila y columna se alternan entre blanco y negro.
	bool par=false;
	for(int i=90; i<=390; i = i+20){	
		// Comprobamos si los puntos impares de la segunda fila y columna son blancos.
		if(!par && (mat.at<uchar>(30,i)  == 0 || mat.at<uchar>(i,30) == 0)){
			// Si no se cumple, no es un codigo.
			cvReleaseImage(&grayframe);
			return false;
		}
		else if(par && (mat.at<uchar>(30,i)  == 255 || mat.at<uchar>(i,30) == 255)){
			// Comprobamos si los puntos pares de la segunda fila y columna son negros.
			cvReleaseImage(&grayframe);
			// Si no se cumple, no es un codigo.
			return false;
		}
		// Comprobamos si todos los puntos de la tercera fila y columna son blancos.
		if(mat.at<uchar>(50,i)  == 0 || mat.at<uchar>(i,50) == 0){
			// Si no se cumple, no es un codigo.
			cvReleaseImage(&grayframe);
			return false;
		}
		par = !par;
	}

	//Comprobamos que el en el cuadrado de referencia todos los puntos sean negros.
	for(int i=30; i<=70; i=i+20){
		if (mat.at<uchar>(30,i)  == 255 || mat.at<uchar>(50,i) == 255 || mat.at<uchar>(70,i) == 255){
			// Si no se cumple, no es un codigo.
			cvReleaseImage(&grayframe);
			return false;
		}
	}

	//Por ultimo comprobamos que los cuadrados que rodean al cuadrado de referencia sean blancos.
	if(mat.at<uchar>(90,50) == 0 || mat.at<uchar>(90,70)  == 0 || mat.at<uchar>(90,90)  == 0 || mat.at<uchar>(70,90)  == 0 ||  mat.at<uchar>(50,90)  == 0){
		// Si no se cumple, no es un codigo.
		cvReleaseImage(&grayframe);
		return false;
	}

	//Si no se cumple ninguna de las condiciones anteriores, significa que hemos encontrado un codigo.
	cvReleaseImage(&grayframe);
	return true;
}

/**
/* @brief Rota una imagen.
/* @param source Matriz inicial.
/* @param angle Angulo a rotar.
/* @param source center centro de giro.
/* @return matriz rotada
**/
Mat rotateImage(const Mat& source, double angle, Point2f center){
    Mat rot_mat = getRotationMatrix2D(center, angle, 1.0);
    Mat dst;
    warpAffine(source, dst, rot_mat, source.size());
    return dst;
}

/**
/* @brief Esta funcion lee un codigo y lo muestra por pantalla. Ademas devuelve la cadena de caracteres leida.
/* @param code Matriz con el codigo.
/* @return cadena de caracteres leida.
**/
char *readCode(const Mat *code){

	//Binarizamos y umbralizamos la imagen para obtener mejores resultados.
	IplImage aux(*code);
	cvThreshold(&aux,&aux,128,255,CV_THRESH_BINARY); 	
	IplImage* grayframe = cvCreateImage(cvGetSize(&aux), IPL_DEPTH_8U, 1);
	cvCvtColor(&aux, grayframe, CV_RGB2GRAY);
	Mat mat(grayframe,true);

	//Reservamos memoria para la cadena de caracteres.
	char code_str[47]; 
	for(int i=0; i<=46; i++){
		// Almacenamos cada uno de los caracteres en el array.
		code_str[i] = readChar(&mat, i);
		// Si el caracter no es NULL lo imprimimos por pantalla.
		if(code_str[i] != NULL) cout<<code_str[i];
		//else cout<<"  letra: NULL"<<endl;
	}
	cout<<endl;

	cvReleaseImage(&grayframe);
	return code_str;
}

/**
/* @brief Lee el caracter en una posicion determinada del codigo.
/* @param code Matriz con el codigo.
/* @param pos Posicion del codigo que deseamos leer.
/* @return caracter leido.
**/
char readChar(const Mat* code, int pos){
	
	// Si la posicion esta entre 0 y 39, el punto de referencia empieza en 110,70 y recorre todo el codigo
	// horizontalmente cada 60 pixels y verticalmente cada 40.
	if(pos<=39){
		int cont = 0;
		for(int y = 0; y<=7; y++){
			for(int x = 0; x<=4; x++){
				if(cont == pos){
					return readLetter(code,(x*60+110),(y*40+70)); 
				}
				cont++;
			}
		}

	}
	// Si la posicion esta entre 40 y 44, el punto de referencia empieza en 70,110 y recorre todo el codigo
	// verticalmente cada 60 pixels.
	else if(pos>=40 && pos <= 44){
		int cont = 40;
		for(int y = 0; y<=4; y++){
				if(cont == pos){
					return readLetter(code,70,(y*60+110)); //Coordenadas normales
				}
				cont++;
		}
	}
	//Si es la posicion 45, las coordenadas del primer punto son 110,390.
	else if(pos==45){
		return readLetter(code,110,390);
	}
	//Si es la posicion 45, las coordenadas del primer punto son 230,390.
	else if(pos==46){
		return readLetter(code,230,390);
	}
}

/**
/* @brief Lee una letra a partir de unas coordenadas de referencia.
/* @param code Matriz con el codigo.
/* @param x,y coordenadas de referencia
/* @return caracter leido.
**/
char readLetter(const Mat* code, int x, int y) {
	
	int num = 0;
	// Asignamos un valor potencia de dos a cada pixel para obtener el numero convertido de binario a decimal.
	// Segun las coordenadas podremos saber que tipo de caracter es y como tiene que ser leido entre los 3 tipos
	// que tenemos.
	// Tengase en cuenta que las coordenadas en una matriz del tipo Mat estan cambiadas con respecto a las coordenadas
	// de los puntos. En una mat, la coordenada y corresponde al eje horizontal y la x al vertical, siendo el inicio el punto
	// de la esquina superior izquierda.
	if(y==390){
		if(code->at<uchar>(y,x)==0){ num = num + 32;}
		if(code->at<uchar>(y,x+20)==0){ num = num + 16;}
		if(code->at<uchar>(y,x+40)==0){ num = num + 8;}
		if(code->at<uchar>(y,x+60)==0){ num = num + 4;}
		if(code->at<uchar>(y,x+80)==0){ num = num + 2;}
		if(code->at<uchar>(y,x+100)==0){ num = num + 1;}	
	}
	else if (x==70){
		if(code->at<uchar>(y,x)==0){ num = num + 32;}
		if(code->at<uchar>(y,x+20)==0){ num = num + 16;}
		if(code->at<uchar>(y+20,x)==0){ num = num + 8;}
		if(code->at<uchar>(y+20,x+20)==0){ num = num + 4;}
		if(code->at<uchar>(y+40,x)==0){ num = num + 2;}
		if(code->at<uchar>(y+40,x+20)==0){ num = num + 1;}
	}
	else{
		if(code->at<uchar>(y,x)==0){ num = num + 32;}
		if(code->at<uchar>(y,x+20)==0) {num = num + 16;}	
		if(code->at<uchar>(y,x+40)==0) {num = num + 8;}
		if(code->at<uchar>(y+20,x)==0) {num = num + 4;}
		if(code->at<uchar>(y+20,x+20)==0) {num = num + 2;}
		if(code->at<uchar>(y+20,x+40)==0) {num = num + 1;}
	}

	//Una vez obtenido el valor numerico se procede a decodificarlo para obtener el caracter.
	// Si el numero es 0, el caracter es null.

	if(num == 0) {return NULL;}
	//Si el numero esta entre 1 y 26, se trata de una letra minuscula. Se devuelve 96 + el numero 
	//leido, ya las letras minusculas en el codigo ASCII estan entre el 97 y el 122.
	if( num >= 1 && num <= 26) {return (96 + num);}

	//Si el numero esta entre 27 y 52, se trata de una letra mayuscula. Se devuelve 38 + el numero 
	//leido, ya las letras mayusculas en el codigo ASCII estan entre el 65 y el 90.
	if( num >= 27 && num <= 52){return (38 + num);}

	//Si es cualquier otro numero se devuelve el valor del codigo ascii del caracter leido.
	if(num==53) {return(32);}
	if(num==54) {return(63);}
	if(num==55)	{return(33);}
	if(num==56) {return(46);}
	if(num==57) {return(44);}
	if(num==58) {return(40);}
	if(num==59) {return(41);}
	if(num==60) {return(47);}
	if(num==61) {return(34);}
	if(num==62) {return(59);}
	if(num==63) {return(58);}
	//Si no corresponde a ninguna de las anteriores devuelve un 0.
	//En teoria es imposible llegar hasta aqui.
	return 0;
}
