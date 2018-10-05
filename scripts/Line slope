##Line slope v4e=name
##Segment_length_multiplier=number 5
##Line_layer=vector line
##DTM=raster

from qgis.core import *
from qgis.utils import *
from PyQt4.QtGui import *
from PyQt4.QtCore import *
import processing
from osgeo import gdal

while len(QgsMapLayerRegistry.instance().mapLayersByName('SplitPoints')) > 0:
    layer = QgsMapLayerRegistry.instance().mapLayersByName('SplitPoints')[0]
    QgsMapLayerRegistry.instance().removeMapLayer(layer.id())

while len(QgsMapLayerRegistry.instance().mapLayersByName('Intersection')) > 0:
    layer = QgsMapLayerRegistry.instance().mapLayersByName('Intersection')[0]
    QgsMapLayerRegistry.instance().removeMapLayer(layer.id())

if (Segment_length_multiplier < 4):
    QMessageBox.information(None, "Note", "The value of the Segment Length Multiplier must be greater than 4. We advise a value between 4 and 10. Ending script.")
else:
    theLayerActive = processing.getObject(Line_layer)
    iface.setActiveLayer(theLayerActive)

    layerCRS = theLayerActive.crs()
    lyrCRS =layerCRS.authid()
    if lyrCRS=="EPSG:4326":
        QMessageBox.information(None, "Note", "The line layer is in geographic coordinates. Reproject this layer in a projected coordinate system before proceeding. Ending script.")
    else:
        if theLayerActive.wkbType()==QGis.WKBMultiLineString:
            QMessageBox.information(None, "Note", "MultiLineString active layer!. Save the layer as a shapefile. Ending script."  )
        else:
            theRasterLayer = gdal.Open(DTM)
            bands = theRasterLayer.RasterCount
            if not bands==1:
                QMessageBox.information(None, "Note", "Multiband DTM. Choose a DTM with only one band. Ending script." )
            else:
                progress.setPercentage(0)
                progress.setText("Calculating split points...")
                geoTransf =theRasterLayer.GetGeoTransform()
                pixelSizeX = abs(geoTransf[1])
                pixelSizeY = abs(geoTransf[5])
                pixelSize = ((pixelSizeX + pixelSizeY) / 2)
                Segment_length = (Segment_length_multiplier * pixelSize)
                points = []
                progress.setPercentage(25)
                time.sleep(2)
                for feat in theLayerActive.getFeatures():
                    geom = feat.geometry()
                    length = geom.length()
                    iter = Segment_length
                    while iter <= (float(length)-float(Segment_length)):
                        pt = geom.interpolate(iter).exportToWkt()
                        points.append(pt)
                        iter += Segment_length
                epsg = theLayerActive.crs().authid()
                uri = "Point?crs=" + epsg + "&field=id:integer""&index=yes"
                mem_layer = QgsVectorLayer(uri,'SplitPoints', 'memory')
                prov = mem_layer.dataProvider()
                feats = [ QgsFeature() for i in range(len(points)) ]
                for i, feat in enumerate(feats):
                    feat.setAttributes([i])
                    feat.setGeometry(QgsGeometry.fromWkt(points[i]))
                prov.addFeatures(feats)
                QgsMapLayerRegistry.instance().addMapLayer(mem_layer)
                progress.setPercentage(50)
                progress.setText("Building segmented line layer named 'Intersection'...")
                time.sleep(2)
                Intersection=processing.runandload('saga:splitlinesatpoints', theLayerActive,mem_layer,1,0.001,None)
                if len(QgsMapLayerRegistry.instance().mapLayersByName('Intersection')) == 0:
                    QMessageBox.information(None, "Error!", "Segmented line layer named 'Intersection' not created. Ending script." )
                    while len(QgsMapLayerRegistry.instance().mapLayersByName('SplitPoints')) > 0:
                        layer = QgsMapLayerRegistry.instance().mapLayersByName('SplitPoints')[0]
                        QgsMapLayerRegistry.instance().removeMapLayer(layer.id())
                else:
                    theactivelayer =  iface.activeLayer()
                    theactivelayer.startEditing()
                    theFields = []
                    for field in theactivelayer.fields():
                        idx = theactivelayer.fieldNameIndex('Slope')
                        theFields.append(idx)
                        theactivelayer.deleteAttributes(theFields)
                    theactivelayer.updateFields()
                    theactivelayer.commitChanges()
                    theactivelayer.startEditing()
                    theactivelayer.dataProvider().addAttributes([QgsField("Slope",  QVariant.Double, "double", 15, 3)])
                    theactivelayer.updateFields()
                    theactivelayer.commitChanges()
                    progress.setPercentage(100)
                    time.sleep(2)
                    progress.setPercentage(0)
                    progress.setText("Calculating segmented line layer slopes...")
                    count = theactivelayer.featureCount()
                    bandeira1 = [0]
                    i = 0
                    features = theactivelayer.getFeatures()
                    for feature in features:
                        i = i + 1
                        percent = (i/float(count)) * 100
                        progress.setPercentage(percent)
                        geom = feature.geometry()
                        feat_geom = geom.asPolyline()
                        x_st=feat_geom[0][0]
                        y_st=feat_geom[0][1]
                        x_end=feat_geom[-1][0]
                        y_end=feat_geom[-1][1]
                        geometryV2 = feature.geometry().geometry()
                        rasterx = int((x_st - geoTransf[0]) / geoTransf[1])
                        rastery = int((y_st - geoTransf[3]) / geoTransf[5])
                        elevationStart=theRasterLayer.GetRasterBand(1).ReadAsArray(rasterx,rastery, 1, 1)
                        rasterx = int((x_end - geoTransf[0]) / geoTransf[1])
                        rastery = int((y_end - geoTransf[3]) / geoTransf[5])
                        elevationEnd=theRasterLayer.GetRasterBand(1).ReadAsArray(rasterx,rastery, 1, 1) 
                        geom = feature.geometry()
                        Comprimento = geom.length()
                        if (elevationStart < -99) : 
                            bandeira1=[1]
                            continue
                        elif (elevationEnd < -99) :
                            bandeira1=[1]
                            continue
                        elif (Comprimento < (pixelSize * 3)) :
                            bandeira1=[1]
                            continue
                        elevation = int(abs((elevationStart - elevationEnd)*10000))
                        theactivelayer.startEditing()
                        idx=theactivelayer.fieldNameIndex('Slope')
                        e=QgsExpression( ''+str(elevation)+'/($length * 100 )')
                        e.prepare(theactivelayer.pendingFields())
                        feature[idx]=e.evaluate(feature)
                        theactivelayer.updateFeature(feature)
                        theactivelayer.commitChanges()
                    if bandeira1[0] == 1:
                        QMessageBox.information(None, "NULL slope values","Two cases can be admited: 1-some (or all) start or end points of segments lay in zones of the DTM with NO DATA values, or, 2-some elements of the line layer are shorter than 3 times the size of the DTM grid. In these cases the slope values were left as NULL.")
                    else:
                        QMessageBox.information(None, "", "Script finished!")
#
while len(QgsMapLayerRegistry.instance().mapLayersByName('SplitPoints')) > 0:
    layer = QgsMapLayerRegistry.instance().mapLayersByName('SplitPoints')[0]
    QgsMapLayerRegistry.instance().removeMapLayer(layer.id())
#  Sobral Almeida, Setembro/2018

